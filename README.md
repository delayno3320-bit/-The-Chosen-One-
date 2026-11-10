#"The Chosen One"

# StickBuddy_Paint.ps1 - a harmless desktop "virus" prank for Windows
# "The Chosen One" (Alan Becker style) draws himself on your screen, waves, then just wanders around the bottom
# of your screen doing his own thing. Leave him alone and he leaves you alone.
# ATTACK him (grab him, drag him, THROW him) and he lights his hands on fire and FLIES to your mouse pointer - head first,
# flames streaming out behind him - then hovers in front of you and zaps you with his EYE LASERS. Just for fun the lasers
# also slice your front window in half (only an illusion: nothing really breaks, it all comes back a moment later).
# Now and then ($RetroChance) the lasers do something else instead: he zaps your desktop back to 2001. For 2 minutes you get
# a Windows XP look-alike - a Bliss-style hill wallpaper, a blue taskbar with a green Start button, an open Start menu - and
# then he changes it all back. It is a costume drawn on top of your real desktop (plus a temporary wallpaper swap that is
# never saved, so a restart always brings your own wallpaper back). Nothing is installed or changed for good.
# After that he rides your pointer. While he rides it, Shift+drag grabs him off. Press ESC (or close this window) and he leaves.
#
# MS PAINT - open Paint and he flies into the canvas and treats it as his stage:
#   * your drawings are SOLID: he can't walk, fall or fly through the lines you draw. He can stand on them and hop onto them.
#   * rub him out: scribble over an arm, a leg or his head and that part disappears (it grows back after ~15 s).
#     No arms = no fire = no flying, so he has to walk (or crawl) after you.
#   * trap him: draw a closed shape around him (freehand loop, rectangle or ellipse). Trapped and angry, he burns his
#     way out with his lasers (they burn holes in your lines) - or you rub out part of the wall.
#   * he stops to watch while you draw (so you can catch him), and if both arms are gone he falls out of the sky.
#   * he fights back: he hurls paint balls (and white-out) at your pointer. Splats cover your drawing (scrub them away), and with
#     $MouseNudge on, a hit shoves your pointer a little (never while a mouse button is held down).
#   Paint itself is never touched: everything he does is drawn on top of it. Nothing is saved or sent.
# It installs nothing, spreads nowhere, and does not run at startup.

if ([Threading.Thread]::CurrentThread.ApartmentState -ne 'STA') {
    & powershell.exe -STA -NoProfile -ExecutionPolicy Bypass -File $PSCommandPath
    exit
}
Add-Type -AssemblyName PresentationFramework, PresentationCore, WindowsBase, System.Windows.Forms
if (-not ('NativeBits' -as [type])) {
    Add-Type -TypeDefinition @"
using System;
using System.Runtime.InteropServices;
public static class NativeBits {
    [DllImport("user32.dll")] public static extern int GetWindowLong(IntPtr h, int i);
    [DllImport("user32.dll")] public static extern int SetWindowLong(IntPtr h, int i, int v);
    [DllImport("user32.dll")] public static extern short GetAsyncKeyState(int k);
}
"@
}

# ---------- Paint support: finds the Paint window and looks at the screen to see what you drew ----------
# Optional: if this part can't be built on your PC, everything else in StickBuddy still works.
$paintCode = @'
using System;
using System.Diagnostics;
using System.Drawing;
using System.Drawing.Imaging;
using System.Runtime.InteropServices;
using System.Text;

public class PaintWin { public IntPtr H; public int L, T, R, B; }

public static class PaintBits {
    delegate bool EnumProc(IntPtr h, IntPtr l);
    [StructLayout(LayoutKind.Sequential)] struct RECT { public int L, T, R, B; }
    [DllImport("user32.dll")] static extern bool EnumWindows(EnumProc f, IntPtr l);
    [DllImport("user32.dll")] static extern bool IsWindowVisible(IntPtr h);
    [DllImport("user32.dll")] static extern bool IsIconic(IntPtr h);
    [DllImport("user32.dll")] static extern bool GetWindowRect(IntPtr h, out RECT r);
    [DllImport("user32.dll", CharSet = CharSet.Unicode)] static extern int GetWindowText(IntPtr h, StringBuilder s, int n);
    [DllImport("user32.dll", CharSet = CharSet.Unicode)] static extern int GetClassName(IntPtr h, StringBuilder s, int n);
    [DllImport("user32.dll")] static extern uint GetWindowThreadProcessId(IntPtr h, out uint pid);
    [DllImport("user32.dll")] static extern IntPtr GetForegroundWindow();
    [DllImport("user32.dll")] static extern bool SetCursorPos(int x, int y);

    static PaintWin Make(IntPtr h) {
        RECT r;
        if (!GetWindowRect(h, out r)) return null;
        PaintWin w = new PaintWin();
        w.H = h; w.L = r.L; w.T = r.T; w.R = r.R; w.B = r.B;
        return w;
    }

    // the front-most open Paint window, or null
    public static PaintWin Find() {
        PaintWin found = null;
        EnumWindows(delegate(IntPtr h, IntPtr l) {
            if (!IsWindowVisible(h) || IsIconic(h)) return true;
            StringBuilder sb = new StringBuilder(256);
            GetClassName(h, sb, 256);
            bool ok = (sb.ToString() == "MSPaintApp");
            if (!ok) {
                sb.Length = 0;
                GetWindowText(h, sb, 256);
                if (sb.ToString().IndexOf("Paint", StringComparison.OrdinalIgnoreCase) >= 0) {
                    uint pid = 0;
                    GetWindowThreadProcessId(h, out pid);
                    try {
                        string pn = Process.GetProcessById((int)pid).ProcessName.ToLower();
                        ok = (pn == "mspaint" || pn == "paintapp" || pn == "paint");
                    } catch (Exception) { }
                }
            }
            if (!ok) return true;
            PaintWin w = Make(h);
            if (w == null || w.R - w.L < 300 || w.B - w.T < 250) return true;
            found = w;
            return false;
        }, IntPtr.Zero);
        return found;
    }

    public static PaintWin RectOf(IntPtr h) {
        if (h == IntPtr.Zero || !IsWindowVisible(h) || IsIconic(h)) return null;
        return Make(h);
    }
    public static bool IsForeground(IntPtr h) { return GetForegroundWindow() == h; }
    public static bool NudgeCursor(int x, int y) { return SetCursorPos(x, y); }

    // for every sample point: is there anything that is not (near) white within r pixels? One screen grab for all points.
    public static bool[] InkFlags(int[] xs, int[] ys, int r) {
        try {
            int n = xs.Length;
            bool[] res = new bool[n];
            if (n == 0) return res;
            int minX = int.MaxValue, minY = int.MaxValue, maxX = int.MinValue, maxY = int.MinValue;
            for (int i = 0; i < n; i++) {
                if (xs[i] < minX) minX = xs[i];
                if (xs[i] > maxX) maxX = xs[i];
                if (ys[i] < minY) minY = ys[i];
                if (ys[i] > maxY) maxY = ys[i];
            }
            minX -= r + 1; minY -= r + 1; maxX += r + 1; maxY += r + 1;
            int w = maxX - minX + 1, h = maxY - minY + 1;
            if (w < 1 || h < 1 || w > 6000 || h > 4000) return null;
            using (Bitmap bmp = new Bitmap(w, h, PixelFormat.Format32bppRgb)) {
                using (Graphics g = Graphics.FromImage(bmp)) {
                    g.CopyFromScreen(minX, minY, 0, 0, new Size(w, h));
                }
                BitmapData bd = bmp.LockBits(new Rectangle(0, 0, w, h), ImageLockMode.ReadOnly, PixelFormat.Format32bppRgb);
                try {
                    int stride = bd.Stride;
                    byte[] buf = new byte[stride * h];
                    Marshal.Copy(bd.Scan0, buf, 0, buf.Length);
                    for (int i = 0; i < n; i++) {
                        int cx = xs[i] - minX, cy = ys[i] - minY;
                        bool hit = false;
                        for (int yy = cy - r; yy <= cy + r && !hit; yy++) {
                            if (yy < 0 || yy >= h) continue;
                            for (int xx = cx - r; xx <= cx + r; xx++) {
                                if (xx < 0 || xx >= w) continue;
                                int o = yy * stride + xx * 4;
                                if (buf[o] < 200 || buf[o + 1] < 200 || buf[o + 2] < 200) { hit = true; break; }
                            }
                        }
                        res[i] = hit;
                    }
                } finally { bmp.UnlockBits(bd); }
            }
            return res;
        } catch (Exception) { return null; }
    }
}
'@
$paintOk = $false
try {
    if (-not ('PaintBits' -as [type])) { Add-Type -TypeDefinition $paintCode -ReferencedAssemblies System.Drawing -ErrorAction Stop }
    $paintOk = $true
} catch {
    Write-Host 'StickBuddy: the Paint features are not available on this PC (everything else still works).' -ForegroundColor DarkGray
}

# ---------- extras: a picture of your drawing (so the lines are solid), the front window (so he can slice it) ----------
# Also optional: if this part can't be built, StickBuddy just walks through drawings and cannot slice windows.
$xCode = @'
using System;
using System.Drawing;
using System.Drawing.Imaging;
using System.Runtime.InteropServices;
using System.Text;

// a picture of your drawing: one cell = Cell x Cell screen pixels, 1 = there is a line in it
public class InkMap {
    public byte[] Cells;
    public int CW, CH, Cell, X0, Y0;       // the grid: size in cells, cell size, screen position of its top-left corner
    public int CL, CT, CR, CB;             // the canvas: only things inside it count as drawings
    public int[] Holes = new int[0];       // x, y, radius triples where his lasers burned the drawing away

    public bool Solid(int px, int py) {
        if (px < CL || px >= CR || py < CT || py >= CB) return false;
        if (px < X0 || py < Y0) return false;
        int cx = (px - X0) / Cell, cy = (py - Y0) / Cell;
        if (cx >= CW || cy >= CH) return false;
        if (Cells[cy * CW + cx] == 0) return false;
        for (int i = 0; i + 2 < Holes.Length; i += 3) {
            int dx = px - Holes[i], dy = py - Holes[i + 1], r = Holes[i + 2];
            if (dx * dx + dy * dy <= r * r) return false;
        }
        return true;
    }

    // does a body with its feet at (fx, fy) touch a drawing? hw = half width, ht = height, lo = lowest height that counts (so he can step over tiny things)
    public bool BodyHit(int fx, int fy, int hw, int ht, int lo) {
        for (int up = lo; up <= ht; up += 8) {
            int y = fy - up;
            if (Solid(fx - hw, y) || Solid(fx, y) || Solid(fx + hw, y)) return true;
        }
        return false;
    }

    // the first solid row at or below fromY (looking straight down), or -1
    public int GroundY(int px, int fromY, int maxDist, int hw) {
        for (int d = 0; d <= maxDist; d += 2) {
            int y = fromY + d;
            if (Solid(px, y) || Solid(px - hw, y) || Solid(px + hw, y)) return y;
        }
        return -1;
    }

    // walks along a line and says how far it gets before touching a drawing (-1 = nothing in the way)
    public int RayHit(int x0, int y0, int x1, int y1, int step) {
        double dx = x1 - x0, dy = y1 - y0;
        double len = Math.Sqrt(dx * dx + dy * dy);
        if (len < 2) return -1;
        double ux = dx / len, uy = dy / len;
        for (double d = step; d <= len; d += step) {
            if (Solid((int)(x0 + ux * d), (int)(y0 + uy * d))) return (int)d;
        }
        return -1;
    }
}

public static class XBits {
    [StructLayout(LayoutKind.Sequential)] struct RECT { public int L, T, R, B; }
    [DllImport("user32.dll")] static extern IntPtr GetForegroundWindow();
    [DllImport("user32.dll")] static extern bool IsWindowVisible(IntPtr h);
    [DllImport("user32.dll")] static extern bool IsIconic(IntPtr h);
    [DllImport("user32.dll")] static extern bool GetWindowRect(IntPtr h, out RECT r);
    [DllImport("user32.dll", CharSet = CharSet.Unicode)] static extern int GetClassName(IntPtr h, StringBuilder s, int n);
    [DllImport("dwmapi.dll")] static extern int DwmGetWindowAttribute(IntPtr h, int attr, out RECT r, int size);
    [DllImport("gdi32.dll")] public static extern bool DeleteObject(IntPtr h);

    // the window you are using right now: {left, top, right, bottom} in screen pixels, or null (desktop, taskbar, tiny windows...)
    public static int[] ForeRect() {
        try {
            IntPtr h = GetForegroundWindow();
            if (h == IntPtr.Zero || !IsWindowVisible(h) || IsIconic(h)) return null;
            StringBuilder sb = new StringBuilder(256);
            GetClassName(h, sb, 256);
            string cn = sb.ToString();
            if (cn == "Progman" || cn == "WorkerW" || cn == "Shell_TrayWnd" || cn == "Shell_SecondaryTrayWnd") return null;
            RECT r;
            if (DwmGetWindowAttribute(h, 9, out r, Marshal.SizeOf(typeof(RECT))) != 0) {
                if (!GetWindowRect(h, out r)) return null;
            }
            if (r.R - r.L < 260 || r.B - r.T < 160) return null;
            return new int[] { r.L, r.T, r.R, r.B };
        } catch (Exception) { return null; }
    }

    // the longest stretch of non-zero entries (a gap of up to maxGap zeros is allowed inside it)
    static void LongestRun(int[] k, int maxGap, out int bestS, out int bestE) {
        bestS = 0; bestE = -1;
        int s = -1, last = -1;
        for (int i = 0; i <= k.Length; i++) {
            bool ok = i < k.Length && k[i] != 0;
            if (ok) {
                if (s < 0) s = i;
                last = i;
            } else if (s >= 0 && (i >= k.Length || i - last > maxGap)) {
                if (last - s > bestE - bestS) { bestS = s; bestE = last; }
                s = -1;
            }
        }
    }

    // looks at a window from a distance and finds the big white area (the Paint canvas): {left, top, right, bottom} in screen pixels, or null
    public static int[] FindCanvas(int x, int y, int w, int h) {
        try {
            if (w < 200 || h < 200 || w > 7000 || h > 4000) return null;
            using (Bitmap bmp = new Bitmap(w, h, PixelFormat.Format32bppRgb)) {
                using (Graphics g = Graphics.FromImage(bmp)) {
                    g.CopyFromScreen(x, y, 0, 0, new Size(w, h));
                }
                BitmapData bd = bmp.LockBits(new Rectangle(0, 0, w, h), ImageLockMode.ReadOnly, PixelFormat.Format32bppRgb);
                try {
                    int stride = bd.Stride;
                    int step = 4;
                    int nr = h / step, nc = w / step;
                    // rows first: 1 = mostly white, 2 = mostly dark (a long black line), 0 = anything else (menus, margins...)
                    int[] rowKind = new int[nr];
                    for (int j = 0; j < nr; j++) {
                        int white = 0, dark = 0;
                        for (int i = 0; i < nc; i++) {
                            int px = Marshal.ReadInt32(bd.Scan0, j * step * stride + i * step * 4);
                            int bb = px & 255, gg = (px >> 8) & 255, rr = (px >> 16) & 255;
                            if (bb > 251 && gg > 251 && rr > 251) white++;
                            else if (bb < 120 && gg < 120 && rr < 120) dark++;
                        }
                        if (white * 100 >= nc * 25) rowKind[j] = 1;
                        else if (dark * 100 >= nc * 40) rowKind[j] = 2;
                    }
                    int r0 = 0, r1 = -1;
                    LongestRun(rowKind, 1, out r0, out r1);
                    int nrows = r1 - r0 + 1;
                    if (nrows < 40) return null;
                    // then the columns, looking only at those rows
                    int[] colKind = new int[nc];
                    for (int i = 0; i < nc; i++) {
                        int white = 0, dark = 0;
                        for (int j = r0; j <= r1; j++) {
                            int px = Marshal.ReadInt32(bd.Scan0, j * step * stride + i * step * 4);
                            int bb = px & 255, gg = (px >> 8) & 255, rr = (px >> 16) & 255;
                            if (bb > 251 && gg > 251 && rr > 251) white++;
                            else if (bb < 120 && gg < 120 && rr < 120) dark++;
                        }
                        if (white * 100 >= nrows * 12) colKind[i] = 1;
                        else if (dark * 100 >= nrows * 60) colKind[i] = 2;
                    }
                    int c0 = 0, c1 = -1;
                    LongestRun(colKind, 1, out c0, out c1);
                    if (c1 - c0 < 40) return null;
                    return new int[] { x + c0 * step, y + r0 * step, x + (c1 + 1) * step, y + (r1 + 1) * step };
                } finally { bmp.UnlockBits(bd); }
            }
        } catch (Exception) { return null; }
    }

    // a picture of the drawing inside the canvas, around a spot: (x, y, w, h) is the area to look at, (cl, ct, cr, cb) the canvas, all in screen pixels
    public static InkMap Scan(int x, int y, int w, int h, int cell, int cl, int ct, int cr, int cb) {
        try {
            int x1 = Math.Max(x, cl), y1 = Math.Max(y, ct), x2 = Math.Min(x + w, cr), y2 = Math.Min(y + h, cb);
            int gw = x2 - x1, gh = y2 - y1;
            if (gw < 8 || gh < 8 || gw > 6000 || gh > 4000) return null;
            using (Bitmap bmp = new Bitmap(gw, gh, PixelFormat.Format32bppRgb)) {
                using (Graphics g = Graphics.FromImage(bmp)) {
                    g.CopyFromScreen(x1, y1, 0, 0, new Size(gw, gh));
                }
                BitmapData bd = bmp.LockBits(new Rectangle(0, 0, gw, gh), ImageLockMode.ReadOnly, PixelFormat.Format32bppRgb);
                try {
                    int stride = bd.Stride;
                    byte[] buf = new byte[stride * gh];
                    Marshal.Copy(bd.Scan0, buf, 0, buf.Length);
                    InkMap m = new InkMap();
                    m.Cell = cell; m.X0 = x1; m.Y0 = y1;
                    m.CW = (gw + cell - 1) / cell; m.CH = (gh + cell - 1) / cell;
                    m.CL = cl; m.CT = ct; m.CR = cr; m.CB = cb;
                    m.Cells = new byte[m.CW * m.CH];
                    for (int yy = 0; yy < gh; yy++) {
                        int ro = yy * stride, rc = (yy / cell) * m.CW;
                        for (int xx = 0; xx < gw; xx++) {
                            int o = ro + xx * 4;
                            if (buf[o] < 200 || buf[o + 1] < 200 || buf[o + 2] < 200) m.Cells[rc + xx / cell] = 1;
                        }
                    }
                    return m;
                } finally { bmp.UnlockBits(bd); }
            }
        } catch (Exception) { return null; }
    }
}
'@
$xOk = $false
try {
    if (-not ('XBits' -as [type])) { Add-Type -TypeDefinition $xCode -ReferencedAssemblies System.Drawing -ErrorAction Stop }
    $xOk = $true
} catch {
    Write-Host 'StickBuddy: solid drawings and window slicing are not available on this PC (everything else still works).' -ForegroundColor DarkGray
}

# ---------- the 2001 zap: swapping the wallpaper, and listing your open windows for the fake taskbar ----------
# Also optional (and kept in its own block on purpose): if this part can't be built, he just can't do the 2001 zap.
$retroCode = @'
using System;
using System.Collections.Generic;
using System.Runtime.InteropServices;
using System.Text;

public static class RetroBits {
    delegate bool EnumProc(IntPtr h, IntPtr l);
    [DllImport("user32.dll")] static extern bool EnumWindows(EnumProc f, IntPtr l);
    [DllImport("user32.dll")] static extern bool IsWindowVisible(IntPtr h);
    [DllImport("user32.dll")] static extern IntPtr GetWindow(IntPtr h, uint cmd);
    [DllImport("user32.dll")] static extern int GetWindowLong(IntPtr h, int i);
    [DllImport("user32.dll")] static extern IntPtr GetForegroundWindow();
    [DllImport("user32.dll", CharSet = CharSet.Unicode)] static extern int GetWindowText(IntPtr h, StringBuilder s, int n);
    [DllImport("dwmapi.dll")] static extern int DwmGetWindowAttribute(IntPtr h, int attr, out int val, int size);
    [DllImport("user32.dll", CharSet = CharSet.Unicode, EntryPoint = "SystemParametersInfo")] static extern bool GetParam(uint action, uint param, StringBuilder buf, uint ini);
    [DllImport("user32.dll", CharSet = CharSet.Unicode, EntryPoint = "SystemParametersInfo")] static extern bool SetParam(uint action, uint param, string buf, uint ini);

    // the wallpaper that is showing right now (null if Windows won't say)
    public static string GetWallpaper() {
        try {
            StringBuilder sb = new StringBuilder(520);
            if (GetParam(0x73, 520, sb, 0)) return sb.ToString();
        } catch (Exception) { }
        return null;
    }
    // changes the wallpaper for this session only (it is NOT written to your settings, so a restart always brings yours back)
    public static bool SetWallpaper(string path) {
        try { return SetParam(0x14, 0, path, 2); } catch (Exception) { return false; }
    }
    static string TitleOf(IntPtr h) {
        StringBuilder sb = new StringBuilder(256);
        GetWindowText(h, sb, 256);
        return sb.ToString();
    }
    public static string ForeTitle() {
        try { return TitleOf(GetForegroundWindow()); } catch (Exception) { return ""; }
    }
    // the titles of the windows that would have a button on a taskbar (front to back)
    public static string[] Titles(int max) {
        List<string> res = new List<string>();
        try {
            EnumWindows(delegate(IntPtr h, IntPtr l) {
                if (res.Count >= max) return false;
                if (!IsWindowVisible(h)) return true;
                if (GetWindow(h, 4) != IntPtr.Zero) return true;
                int ex = GetWindowLong(h, -20);
                if ((ex & 0x80) != 0 && (ex & 0x40000) == 0) return true;
                int cloaked = 0;
                try { DwmGetWindowAttribute(h, 14, out cloaked, 4); } catch (Exception) { }
                if (cloaked != 0) return true;
                string t = TitleOf(h);
                if (t.Length == 0 || t == "Program Manager") return true;
                res.Add(t);
                return true;
            }, IntPtr.Zero);
        } catch (Exception) { }
        return res.ToArray();
    }
}
'@
$retroOk = $false
try {
    if (-not ('RetroBits' -as [type])) { Add-Type -TypeDefinition $retroCode -ErrorAction Stop }
    $retroOk = $true
} catch {
    Write-Host 'StickBuddy: the 2001 zap is not available on this PC (everything else still works).' -ForegroundColor DarkGray
}

# ---------- settings you can change ----------
$MouseNudge = $true       # a paint-ball hit shoves your pointer a little (only while Paint is in front and no mouse button is down)
$WalkSpeed = 150          # how fast he strolls, in pixels per second
$SolidDrawings = $true    # the lines you draw in Paint are solid: he can't walk, fall or fly through them
$Lasers = $true           # eye lasers when he is angry (they also burn holes in your lines)
$WindowBreak = $true      # the lasers slice your front window in half "just for fun" (only an illusion, nothing really breaks)
$BreakChance = 1.0        # 0 to 1: how often an angry flight ends with a sliced window (1 = every time)
$RetroMode = $true        # he can zap your desktop back to 2001: a Windows XP look-alike drawn on top of your real one
$RetroChance = 0.35       # 0 to 1: how often an angry flight ends with the 2001 zap instead of the sliced window (1 = every time)
$RetroSeconds = 120       # how long the old look stays before he changes it back (120 = 2 minutes)
$RetroWallpaper = $true   # also swap your real wallpaper for a Bliss-style hill (put back afterwards; never saved, so a restart brings yours back)

# ---------- window: small, transparent, always on top ----------
$W = 480; $H = 380; $FX = 240; $FY = 300          # window size and where the feet sit inside it (roomy, so long flames and lasers' glow don't get cut off)
$win = New-Object Windows.Window
$win.WindowStyle = 'None'; $win.AllowsTransparency = $true
$win.Background = [Windows.Media.Brushes]::Transparent
$win.Topmost = $true; $win.ShowInTaskbar = $false; $win.ShowActivated = $false; $win.ResizeMode = 'NoResize'
$win.Width = $W; $win.Height = $H
$win.WindowStartupLocation = 'Manual'; $win.Left = -5000; $win.Top = -5000
$canvas = New-Object Windows.Controls.Canvas
$win.Content = $canvas

# nearly invisible grab zone (alpha 1 still catches the mouse, alpha 0 lets clicks fall through)
$hit = New-Object Windows.Shapes.Rectangle
$hit.Width = 100; $hit.Height = 125
$hit.Fill = [Windows.Media.SolidColorBrush]::new([Windows.Media.Color]::FromArgb(1, 0, 0, 0))
$hit.Cursor = [Windows.Input.Cursors]::Hand
[Windows.Controls.Canvas]::SetLeft($hit, $FX - 50); [Windows.Controls.Canvas]::SetTop($hit, $FY - 125)
[void]$canvas.Children.Add($hit)

# the figure lives in its own canvas so it can tumble when thrown
$fig = New-Object Windows.Controls.Canvas
$fig.Width = $W; $fig.Height = $H; $fig.IsHitTestVisible = $false
$rot = New-Object Windows.Media.RotateTransform
$rot.CenterX = $FX; $rot.CenterY = $FY - 50
$fig.RenderTransform = $rot
[void]$canvas.Children.Add($fig)

$white = [Windows.Media.Brushes]::White; $black = [Windows.Media.Brushes]::Black
function New-Stroke($brush, $thick) {
    $p = New-Object Windows.Shapes.Path
    $p.Stroke = $brush; $p.StrokeThickness = $thick
    $p.StrokeStartLineCap = 'Round'; $p.StrokeEndLineCap = 'Round'; $p.StrokeLineJoin = 'Round'
    [void]$fig.Children.Add($p)
    $p
}
$outline = New-Stroke $white 12                     # thin white rim so he shows on dark backgrounds
$outline.Fill = $white                              # white head, like the original
$inner = New-Stroke $black 8

# fire for the hands
function New-Grad($stops) {
    $b = New-Object Windows.Media.LinearGradientBrush
    $b.StartPoint = [Windows.Point]::new(0.5, 0); $b.EndPoint = [Windows.Point]::new(0.5, 1)
    foreach ($gs in $stops) {
        [void]$b.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb($gs[1], $gs[2], $gs[3], $gs[4]), $gs[0]))
    }
    $b
}
$bOuter = New-Grad @( @(0.0, 255, 255, 215, 70), @(0.4, 255, 255, 175, 25), @(0.8, 200, 255, 120, 10), @(1.0, 0, 255, 100, 10) )
$bMid = New-Grad @( @(0.0, 255, 255, 250, 190), @(0.6, 230, 255, 220, 70), @(1.0, 0, 255, 170, 30) )
$bCore = New-Grad @( @(0.0, 255, 255, 255, 255), @(1.0, 0, 255, 245, 170) )
$fl = @()
foreach ($br in $bOuter, $bOuter, $bMid, $bMid, $bCore, $bCore) {
    $p = New-Object Windows.Shapes.Path
    $p.Fill = $br; $p.Visibility = 'Hidden'
    [void]$fig.Children.Add($p)
    $fl += $p
}

$bubble = New-Object Windows.Controls.Border
$bubble.Background = $white; $bubble.BorderBrush = $black
$bubble.BorderThickness = [Windows.Thickness]::new(2); $bubble.CornerRadius = [Windows.CornerRadius]::new(10)
$bubble.Padding = [Windows.Thickness]::new(8, 2, 8, 3); $bubble.Visibility = 'Hidden'; $bubble.IsHitTestVisible = $false
$tb = New-Object Windows.Controls.TextBlock
$tb.FontSize = 22; $tb.FontWeight = 'Bold'; $tb.Foreground = $black
$bubble.Child = $tb
[void]$canvas.Children.Add($bubble)

# ---------- tiny geometry helpers ----------
function Pt($x, $y) { [Windows.Point]::new($x, $y) }
function Mix($a, $b, $k) { Pt ($a.X + ($b.X - $a.X) * $k) ($a.Y + ($b.Y - $a.Y) * $k) }
function Clamp01($v) { [Math]::Max(0.0, [Math]::Min(1.0, $v)) }
function Clamp($v, $lo, $hi) { [Math]::Max($lo, [Math]::Min($hi, $v)) }
function Dist($a, $b) { [Math]::Sqrt([Math]::Pow($a.X - $b.X, 2) + [Math]::Pow($a.Y - $b.Y, 2)) }
function Brush($r, $g, $b) { [Windows.Media.SolidColorBrush]::new([Windows.Media.Color]::FromRgb($r, $g, $b)) }
function BrushA($a, $r, $g, $b) { [Windows.Media.SolidColorBrush]::new([Windows.Media.Color]::FromArgb($a, $r, $g, $b)) }
# a glowing blob: kind 1 = red (eyes), anything else = white-orange (where a laser hits)
function NewFlareBrush($kind) {
    $rb = New-Object Windows.Media.RadialGradientBrush
    if ($kind -eq 1) { $c0 = @(255, 255, 230, 210); $c1 = @(200, 255, 60, 40); $c2 = @(0, 255, 30, 20) }
    else { $c0 = @(255, 255, 255, 255); $c1 = @(210, 255, 190, 80); $c2 = @(0, 255, 90, 20) }
    [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb($c0[0], $c0[1], $c0[2], $c0[3]), 0.0))
    [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb($c1[0], $c1[1], $c1[2], $c1[3]), 0.45))
    [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb($c2[0], $c2[1], $c2[2], $c2[3]), 1.0))
    $rb
}
# the burnt mark a laser leaves in a line you drew
function NewScorchBrush {
    $rb = New-Object Windows.Media.RadialGradientBrush
    foreach ($st in @( @(0.0, 215, 20, 16, 14), @(0.55, 160, 60, 24, 6), @(0.85, 120, 255, 140, 20), @(1.0, 0, 255, 110, 10) )) {
        [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb($st[1], $st[2], $st[3], $st[4]), $st[0]))
    }
    $rb
}

# smooth curve from $a to $b that bends through $m (so limbs look drawn, not angular)
function Curve($a, $m, $b) {
    $qx = 2 * $m.X - ($a.X + $b.X) / 2; $qy = 2 * $m.Y - ($a.Y + $b.Y) / 2
    $pts = @()
    for ($i = 0; $i -le 8; $i++) {
        $u = $i / 8.0; $v = 1 - $u
        $pts += (Pt ($v * $v * $a.X + 2 * $u * $v * $qx + $u * $u * $b.X) ($v * $v * $a.Y + 2 * $u * $v * $qy + $u * $u * $b.Y))
    }
    , $pts
}

# draws a polyline through $pts, only the first fraction $p of it (so limbs "grow")
function Chain($c, $pts, $p) {
    if ($p -le 0) { return }
    $n = $pts.Count - 1
    $len = $p * $n
    $c.BeginFigure($pts[0], $false, $false)
    for ($i = 1; $i -le $n; $i++) {
        if ($len -ge $i) { $c.LineTo($pts[$i], $true, $true) }
        else {
            $f = $len - ($i - 1)
            if ($f -gt 0) { $c.LineTo((Mix $pts[$i - 1] $pts[$i] $f), $true, $true) }
            break
        }
    }
}
# elbow and hand for one arm (angles measured from "straight down", outward is positive)
function Arm($sh, $sd, $ua, $la) {
    $sd = [double]$sd                                 # (a bare -1 argument arrives as a string)
    $e = Pt ($sh.X + $sd * 22 * [Math]::Sin($ua)) ($sh.Y + 22 * [Math]::Cos($ua))
    $hd = Pt ($e.X + $sd * 22 * [Math]::Sin($la)) ($e.Y + 22 * [Math]::Cos($la))
    , @($e, $hd)
}
# a ragged, spiky flame plume that streams from (x, y) in the direction (dx, dy): down while he hovers, backwards while he flies
function PlumeGeo($x, $y, $len, $wd, $sway, $tt, $seed, $dx, $dy) {
    $g = New-Object Windows.Media.StreamGeometry
    $c = $g.Open()
    $px = $dy; $py = 0 - $dx                           # sideways direction
    $n = 9
    $c.BeginFigure((Pt ($x - $px * $wd) ($y - $py * $wd)), $true, $true)
    for ($i = 1; $i -le $n; $i++) {
        $u = $i / [double]$n
        $jag = 1 + 0.42 * [Math]::Sin($tt * 31 + $i * 2.3 + $seed) * [Math]::Sin($tt * 17 + $i * 1.1 + $seed * 2)
        $al = $len * $u * (1 + 0.1 * [Math]::Sin($tt * 23 + $i * 3.1 + $seed))
        $hw = $wd * [Math]::Pow(1 - $u, 0.75) * $jag
        $la = $sway * $u - $hw
        $c.LineTo((Pt ($x + $dx * $al + $px * $la) ($y + $dy * $al + $py * $la)), $true, $true)
    }
    for ($i = $n - 1; $i -ge 1; $i--) {
        $u = $i / [double]$n
        $jag = 1 + 0.42 * [Math]::Sin($tt * 27 + $i * 3.7 + $seed + 1.7) * [Math]::Sin($tt * 19 + $i * 1.9 + $seed)
        $al = $len * $u * (1 + 0.1 * [Math]::Sin($tt * 21 + $i * 2.7 + $seed))
        $hw = $wd * [Math]::Pow(1 - $u, 0.75) * $jag
        $la = $sway * $u + $hw
        $c.LineTo((Pt ($x + $dx * $al + $px * $la) ($y + $dy * $al + $py * $la)), $true, $true)
    }
    $c.LineTo((Pt ($x + $px * $wd) ($y + $py * $wd)), $true, $true)
    $c.Close()
    $g
}
function Centroid($pts) {
    $sx = 0.0; $sy = 0.0
    foreach ($q in $pts) { $sx += $q.X; $sy += $q.Y }
    Pt ($sx / $pts.Count) ($sy / $pts.Count)
}
function PolyGeo($pts) {
    $g = New-Object Windows.Media.StreamGeometry
    $c = $g.Open()
    $c.BeginFigure($pts[0], $true, $true)
    for ($i = 1; $i -lt $pts.Count; $i++) { $c.LineTo($pts[$i], $true, $false) }
    $c.Close()
    $g
}
# the part of a polygon on one side of a line (through ax,ay with normal nx,ny; $sg = 1 or -1 picks the side)
function ClipHalf($pts, $ax, $ay, $nx, $ny, $sg) {
    $out = New-Object System.Collections.ArrayList
    $n = $pts.Count
    for ($i = 0; $i -lt $n; $i++) {
        $p0 = $pts[$i]; $p1 = $pts[($i + 1) % $n]
        $d0 = $sg * (($p0.X - $ax) * $nx + ($p0.Y - $ay) * $ny)
        $d1 = $sg * (($p1.X - $ax) * $nx + ($p1.Y - $ay) * $ny)
        if ($d0 -ge 0) { [void]$out.Add($p0) }
        if (($d0 -ge 0 -and $d1 -lt 0) -or ($d0 -lt 0 -and $d1 -ge 0)) {
            $f = $d0 / ($d0 - $d1)
            [void]$out.Add((Pt ($p0.X + ($p1.X - $p0.X) * $f) ($p0.Y + ($p1.Y - $p0.Y) * $f)))
        }
    }
    , $out
}

# ---------- state ----------
$wa = [Windows.SystemParameters]::WorkArea
$vl = [Windows.SystemParameters]::VirtualScreenLeft;  $vt = [Windows.SystemParameters]::VirtualScreenTop
$vw = [Windows.SystemParameters]::VirtualScreenWidth; $vh = [Windows.SystemParameters]::VirtualScreenHeight
$rng = New-Object Random
$S = @{
    t = 0.0; last = [DateTime]::UtcNow; phase = 'build'; run = 0.0
    x = ($wa.Right - 320); y = $wa.Bottom              # starts standing on top of the taskbar
    vx = 0.0; vy = 0.0; ang = 0.0; spin = 0.0; to = 0.0; soft = $false; fire = 0.0
    pt = 0.0; sx = 0.0; sy = 0.0; hx = 0.0; hy = 0.0; rideT = 0.0; still = 0.0
    lb = $false; ct = $false; hwnd = [IntPtr]::Zero
    hist = (New-Object System.Collections.ArrayList)
    lc = (Pt 0 0); head = (Pt 0 0); src = $null
    # where he can walk right now: floor, left wall, right wall, ceiling (changes in Paint and inside a trap); sf = stage floor
    cf = $wa.Bottom; cl = ($wa.Left + 35); cr = ($wa.Right - 35); cc = ($wa.Top + 140); sf = $wa.Bottom
    # wandering, attacks and items
    angry = $false; wm = 'idle'; wt = 0.0; wx = 0.0; tw0 = 0.0; hopT = 0.0; nb = 0; nbMax = 2; big = $false; nextItem = 8.0
    # how much of each body part is drawn (0 = rubbed out), when it grows back, and how much it has been scrubbed
    g = @{ head = 1.0; aR = 1.0; aL = 1.0; lR = 1.0; lL = 1.0 }
    er = @{ head = 0.0; aR = 0.0; aL = 0.0; lR = 0.0; lL = 0.0 }
    scr = @{ head = 0.0; aR = 0.0; aL = 0.0; lR = 0.0; lL = 0.0 }
    legs = 2; canFly = $true; jt = @{}
    msg = ''; msgT = 0.0; pc = (Pt 0 0); cu = (Pt 0 0)
    # Paint
    paintOk = $paintOk; pwh = [IntPtr]::Zero; prT = 0.0; pr = $null; pfg = $false; inPaint = $false
    path = $null; pathOn = $false; chk = $null; trap = $null
    fxw = $null; fxc = $null; lzc = $null
    balls = (New-Object System.Collections.ArrayList); splats = (New-Object System.Collections.ArrayList)
    pal = @( (Brush 225 70 60), (Brush 240 160 40), (Brush 70 190 90), (Brush 70 130 230), (Brush 160 90 220), (Brush 235 90 170), (Brush 250 250 250) )
    tgx = 0.0; tgy = 0.0; knockT = 0.0; noFire = $false; errShown = $false
    # solid drawings: a picture of the canvas, where the canvas is, and the places his lasers burned away
    xOk = $xOk; map = $null; mapT = 0.0; mapC = (Pt 0 0); cv = $null; cvT = 0.0; cvKey = 0.0; dx = 1.0; dy = 1.0
    holes = (New-Object System.Collections.ArrayList); holeArr = (New-Object 'int[]' 0); scorch = (NewScorchBrush)
    # looks and moves: which way he faces, landing squash, how long a wall has blocked him, the ground he landed on, the smoothed pose
    face = 1; sq = 0.0; blkT = 0.0; gl = $wa.Bottom
    pose = @{ lean = 0.0; crouch = 0.0; lift = 0.0; lift2 = 0.0; speed = 0.0; ua = 0.8; la = 0.2; ul = 0.8; ll = 0.2; free = 0.0; legA = 0.0; fdx = 0.0; fdy = 1.0; flk = 1.0 }
    # lasers and the sliced window
    lzE = $null; lzOn = $false; lzPow = 0.0; aimX = 0.0; aimY = 0.0; burnT = 0.0; lzFreeT = 0.0; lzHx = 0.0; lzHy = 0.0; lzErr = $false
    hz = ''; hzt = 0.0; doSlice = $false; sl = $null; bp = (Pt 0 0)
    # the 2001 zap: state = off / in / out / on / rin / rout (a flash, the switch, the old look, the flash back, the switch back)
    rtOk = ($retroOk -and $RetroMode); doRetro = $false
    rt = @{ state = 'off'; t0 = 0.0; end = 0.0; flash = $null; tbi = $null; sm = $null; smUntil = 0.0; topT = 0.0; clkT = 0.0; btnT = 0.0
            tbTop = 0.0; clk = $null; btns = (New-Object System.Collections.ArrayList); orig = ''; file = ''; hasWall = $false; saved = $false }
}

function CursorDip {
    $p = [Windows.Forms.Cursor]::Position
    $S.src.CompositionTarget.TransformFromDevice.Transform([Windows.Point]::new($p.X, $p.Y))
}
# while he rides your pointer (or Paint is open) the window must let every click through
function Set-Through($on) {
    if ($S.ct -eq $on -or $S.hwnd -eq [IntPtr]::Zero) { return }
    $st = [NativeBits]::GetWindowLong($S.hwnd, -20)
    if ($on) { $st = $st -bor 0x20 } else { $st = $st -band (-bnot 0x20) }
    [void][NativeBits]::SetWindowLong($S.hwnd, -20, $st)
    $S.ct = $on
}
# putting hands on him counts as an attack: once he has landed again he comes for your pointer
function DoGrab { $S.phase = 'held'; $S.hist.Clear(); $S.ang = 0.0; $S.spin = 0.0; $S.vx = 0.0; $S.vy = 0.0; $S.angry = $true }
function DoRelease {
    $vx = 0.0; $vy = 0.0; $hs = $S.hist
    if ($hs.Count -ge 2) {
        $a = $hs[0]; $b = $hs[$hs.Count - 1]; $d = $b.t - $a.t
        if ($d -gt 0.01 -and ($S.t - $b.t) -lt 0.1) { $vx = ($b.x - $a.x) / $d; $vy = ($b.y - $a.y) / $d }
    }
    $m = [Math]::Sqrt($vx * $vx + $vy * $vy)
    if ($m -gt 2600) { $vx *= 2600 / $m; $vy *= 2600 / $m }
    $S.vx = $vx; $S.vy = $vy; $S.spin = $vx / 350; $S.soft = $false; $S.phase = 'fly'
    if ($null -ne $S.trap -and -not (PolyInside $S.trap.poly $S.x ($S.y - 60))) {          # you dragged him out of his cage
        $S.trap = $null; $S.msg = 'free!'; $S.msgT = $S.t + 1.5
    }
}
$hit.Add_MouseLeftButtonDown({ if ($S.phase -ne 'ride') { DoGrab } })

# ---------- where his body parts are (for scribbling him out) ----------
# joint positions are stored in figure-canvas coordinates; this turns one into a screen position
function ToScr($pt) {
    $dx = $pt.X - $FX; $dy = $pt.Y - ($FY - 50)
    $c = [Math]::Cos($S.ang); $n = [Math]::Sin($S.ang)
    Pt ($S.x + $dx * $c - $dy * $n) ($S.y - 50 + $dx * $n + $dy * $c)
}
function DistSeg($p, $a, $b) {
    $vx = $b.X - $a.X; $vy = $b.Y - $a.Y
    $l2 = $vx * $vx + $vy * $vy
    $u = 0.0
    if ($l2 -gt 0) { $u = [Math]::Max(0.0, [Math]::Min(1.0, (($p.X - $a.X) * $vx + ($p.Y - $a.Y) * $vy) / $l2)) }
    $qx = $a.X + $u * $vx - $p.X; $qy = $a.Y + $u * $vy - $p.Y
    [Math]::Sqrt($qx * $qx + $qy * $qy)
}
function DistPart($k, $cur) {
    $jp = $S.jt[$k]
    if ($k -eq 'head') { return [Math]::Max(0.0, (Dist (ToScr $jp) $cur) - 22) }
    $best = 100000.0
    for ($i = 0; $i -lt $jp.Count - 1; $i++) {
        $d = DistSeg $cur (ToScr $jp[$i]) (ToScr $jp[$i + 1])
        if ($d -lt $best) { $best = $d }
    }
    $best
}
function NearBody($cur) {
    if ($S.jt.Count -eq 0) { return $false }
    foreach ($k in 'head', 'aR', 'aL', 'lR', 'lL') { if ((DistPart $k $cur) -lt 34) { return $true } }
    $ta = $S.jt.torso
    return ((DistSeg $cur (ToScr $ta[0]) (ToScr $ta[1])) -lt 34)
}

# ---------- rubbing out body parts (and growing them back) ----------
function UpdateParts($t, $dt) {
    foreach ($k in 'head', 'aR', 'aL', 'lR', 'lL') {
        if ($S.er[$k] -gt 0) {
            if ($t -ge $S.er[$k]) { $S.er[$k] = 0.0; $S.msg = 'good as new!'; $S.msgT = $t + 1.5 }
            $S.g[$k] = [Math]::Max(0.0, $S.g[$k] - $dt * 6)
        } else { $S.g[$k] = [Math]::Min(1.0, $S.g[$k] + $dt * 1.2) }
    }
    $S.legs = 0
    if ($S.g.lR -gt 0.5) { $S.legs += 1 }
    if ($S.g.lL -gt 0.5) { $S.legs += 1 }
    $S.canFly = (($S.g.aR -gt 0.5) -or ($S.g.aL -gt 0.5))
}
function EraseNow($k) {
    $S.er[$k] = $S.t + 14 + 6 * $rng.NextDouble()
    $S.scr[$k] = 0.0
    $S.angry = $true
    $nm = 'arm'
    if ($k -eq 'head') { $nm = 'head' } elseif ($k -like 'l*') { $nm = 'leg' }
    $S.msg = 'my ' + $nm + '!'; $S.msgT = $S.t + 1.8
}
# scribbling over a part with the mouse button down (eraser, pencil, anything) rubs it out
function ScrubParts($cur, $lb, $dt) {
    $mv = Dist $cur $S.pc
    foreach ($k in 'head', 'aR', 'aL', 'lR', 'lL') {
        if ($S.er[$k] -gt 0 -or $S.g[$k] -lt 0.9) { continue }
        if ($lb -and $mv -gt 0.5 -and (DistPart $k $cur) -lt 18) { $S.scr[$k] += $mv }
        else { $S.scr[$k] = [Math]::Max(0.0, $S.scr[$k] - 80 * $dt) }
        if ($S.scr[$k] -gt 45) { EraseNow $k }
    }
}

# ---------- drawings are solid ----------
function BodyHit($x, $y) {                 # would his body, feet at (x, y), touch a drawing there?
    $mp = $S.map
    if ($null -eq $mp) { return $false }
    return $mp.BodyHit([int]($x * $S.dx), [int]($y * $S.dy), [int](11 * $S.dx), [int](118 * $S.dy), [int](14 * $S.dy))
}
function GroundAt($x, $y) {                # where his feet come to rest here: the floor, or the top of a line you drew
    $g = $S.cf
    $mp = $S.map
    if ($null -ne $mp) {
        $r = $mp.GroundY([int]($x * $S.dx), [int](($y - 12) * $S.dy), [int](900 * $S.dy), [int](4 * $S.dx))
        if ($r -ge 0) { $gy = $r / $S.dy; if ($gy -lt $g) { $g = $gy } }
    }
    return $g
}
# moves him by (mx, my) a few pixels at a time and stops at drawings: 0 = free, +1 = hit something sideways, +2 = hit something above or below
function MoveBody($mx, $my) {
    $mp = $S.map
    if ($null -eq $mp -or -not $SolidDrawings) { $S.x += $mx; $S.y += $my; return 0 }
    $n = [int][Math]::Ceiling([Math]::Max([Math]::Abs($mx), [Math]::Abs($my)) / 6.0)
    if ($n -lt 1) { $n = 1 }
    $ex = $mx / $n; $ey = $my / $n; $res = 0
    $inside = BodyHit $S.x $S.y            # you drew right over him? then let him move, so he can get out
    for ($i = 0; $i -lt $n; $i++) {
        if (-not $inside) {
            if ($ex -ne 0 -and (BodyHit ($S.x + $ex) $S.y)) { $ex = 0.0; $res = $res -bor 1 }
            if ($ey -ne 0 -and (BodyHit $S.x ($S.y + $ey))) { $ey = 0.0; $res = $res -bor 2 }
        }
        $S.x += $ex; $S.y += $ey
    }
    return $res
}

# ---------- wandering ----------
function PickWander {
    $now = $S.t; $r = $rng.NextDouble()
    if ($S.g.head -lt 0.5) { $r = $r * 0.6 }                  # no head: he stumbles around blindly
    if ($r -lt 0.55) {
        $S.wm = 'walk'
        $nx = $S.cl + ($S.cr - $S.cl) * $rng.NextDouble()
        if ([Math]::Abs($nx - $S.x) -lt 80) { $nx = $S.x + 140 * (2 * $rng.Next(2) - 1) }
        $S.wx = [Math]::Max($S.cl, [Math]::Min($S.cr, $nx)); $S.wt = $now + 9
    } elseif ($r -lt 0.78) { $S.wm = 'idle'; $S.wt = $now + 1.5 + 2.5 * $rng.NextDouble()
    } elseif ($r -lt 0.9) { $S.wm = 'wave'; $S.wt = $now + 1.7
    } else { $S.wm = 'hopwind'; $S.hopT = $now + 0.22; $S.wt = $now + 2.0 }
}
# fly (fire jets) towards a point; returns how far away it still is. Head first, arms and legs trailing, flames streaming out behind.
function JetTo($gx, $gy, $dt, $t, $o) {
    $dx = $gx - $S.x; $dy = $gy - $S.y; $d = [Math]::Max(1.0, [Math]::Sqrt($dx * $dx + $dy * $dy))
    $spd = [Math]::Min(650.0, 120 + $d * 2.2); $kk = 1 - [Math]::Exp(-$dt * 4)
    $S.vx += ($dx / $d * $spd - $S.vx) * $kk; $S.vy += ($dy / $d * $spd - $S.vy) * $kk
    $r = MoveBody ($S.vx * $dt) ($S.vy * $dt)
    if ($r -band 1) { $S.vx = 0.0; $S.vy -= 700 * $dt }                 # a drawn wall is in the way: climb over it
    if ($r -band 2) { if ($S.vy -lt 0) { $S.vy = 60.0 } else { $S.vy = 0.0 } }
    $S.run += $dt * 6; $o.speed = 0.15; $o.run = $S.run
    $sp = [Math]::Min(1.0, [Math]::Sqrt($S.vx * $S.vx + $S.vy * $S.vy) / 450.0)
    $lean = [Math]::Max(-1.3, [Math]::Min(1.3, $S.vx / 380.0 * 1.3))   # sideways flight = nearly level, like a superhero
    $o.lean = $lean; $o.free = 1.0; $o.legA = 0 - $lean
    $o.lift = 14 + 4 * [Math]::Sin($t * 9); $o.lift2 = $o.lift
    $kk2 = Clamp01 ([Math]::Abs($S.vx) / 260.0)                         # slow: arms out and down; fast: arms swept back
    $mq = $lean - 0.3 * [Math]::Sign($lean)
    $o.ua = 0.7 * (1 - $kk2) + (0 - $mq) * $kk2; $o.la = 0.35 * (1 - $kk2) + (0 - $mq * 1.15) * $kk2
    $o.ul = 0.7 * (1 - $kk2) + $mq * $kk2;       $o.ll = 0.35 * (1 - $kk2) + ($mq * 1.15) * $kk2
    $fvx = 0 - $S.vx; $fvy = 140 - $S.vy                                # the jets push against the way he is going
    $fn = [Math]::Max(1.0, [Math]::Sqrt($fvx * $fvx + $fvy * $fvy))
    $o.fdx = $fvx / $fn; $o.fdy = $fvy / $fn; $o.flk = 1.0 + 0.9 * $sp
    $d
}
# the pose of his eye-laser stance: upright (or hovering), arms out and down, flames beneath his hands
function LaserPose($o, $hover) {
    $o.lean = 0.14 * $S.face
    $o.ua = 0.75; $o.la = 0.4; $o.ul = 0.75; $o.ll = 0.4
    $o.fdx = 0.0; $o.fdy = 1.0; $o.flk = 1.1
    if ($hover) { $o.free = 1.0; $o.legA = 0.05 * $S.face; $o.crouch = 0.0 }
}

# ---------- polygons (the walls of a trap) ----------
function PolyInside($poly, $x, $y) {
    $inside = $false; $n = $poly.Count; $j = $n - 1
    for ($i = 0; $i -lt $n; $i++) {
        $qa = $poly[$i]; $qb = $poly[$j]
        if ((($qa.Y -gt $y) -ne ($qb.Y -gt $y)) -and ($x -lt ($qb.X - $qa.X) * ($y - $qa.Y) / ($qb.Y - $qa.Y) + $qa.X)) { $inside = -not $inside }
        $j = $i
    }
    $inside
}
# where the outline crosses the vertical line x=$v (axis 'x') or the horizontal line y=$v (axis 'y'), sorted
function PolyCuts($poly, $axis, $v) {
    $cuts = New-Object 'System.Collections.Generic.List[double]'
    $n = $poly.Count
    for ($i = 0; $i -lt $n; $i++) {
        $qa = $poly[$i]; $qb = $poly[($i + 1) % $n]
        if ($axis -eq 'x') { $a1 = $qa.X; $b1 = $qb.X; $a2 = $qa.Y; $b2 = $qb.Y } else { $a1 = $qa.Y; $b1 = $qb.Y; $a2 = $qa.X; $b2 = $qb.X }
        if ((($a1 -le $v) -and ($b1 -gt $v)) -or (($b1 -le $v) -and ($a1 -gt $v))) {
            $cuts.Add($a2 + ($v - $a1) / ($b1 - $a1) * ($b2 - $a2))
        }
    }
    $cuts.Sort()
    , $cuts
}
# the stretch between two crossings that contains $v0, or $null
function PolySpan($cuts, $v0) {
    for ($k = 0; ($k + 1) -lt $cuts.Count; $k += 2) {
        if ($cuts[$k] -le $v0 -and $v0 -le $cuts[$k + 1]) { return , @($cuts[$k], $cuts[$k + 1]) }
    }
    return $null
}
# points every $step pixels along a polyline
function Resample($pts, $step) {
    $out = New-Object System.Collections.ArrayList
    [void]$out.Add($pts[0])
    $acc = 0.0
    for ($i = 1; $i -lt $pts.Count; $i++) {
        $a = $pts[$i - 1]; $b = $pts[$i]
        $seg = Dist $a $b
        if ($seg -le 0) { continue }
        $pos = $step - $acc
        while ($pos -le $seg) {
            $f = $pos / $seg
            [void]$out.Add((Pt ($a.X + ($b.X - $a.X) * $f) ($a.Y + ($b.Y - $a.Y) * $f)))
            $pos += $step
        }
        $acc = $seg - ($pos - $step)
    }
    , $out
}

# ---------- Paint: the window, the stage, traps, paint balls ----------
function EnsureFx {                       # a second, click-through window for paint balls, splats, lasers and the sliced window
    if ($null -ne $S.fxw) { return }
    $w2 = New-Object Windows.Window
    $w2.WindowStyle = 'None'; $w2.AllowsTransparency = $true
    $w2.Background = [Windows.Media.Brushes]::Transparent
    $w2.Topmost = $true; $w2.ShowInTaskbar = $false; $w2.ShowActivated = $false; $w2.ResizeMode = 'NoResize'
    $w2.IsHitTestVisible = $false
    $w2.WindowStartupLocation = 'Manual'
    $w2.Left = $vl; $w2.Top = $vt; $w2.Width = $vw; $w2.Height = $vh
    $grid = New-Object Windows.Controls.Grid
    $c2 = New-Object Windows.Controls.Canvas          # paint balls and splats
    $c3 = New-Object Windows.Controls.Canvas          # lasers, burn marks, the sliced window
    [void]$grid.Children.Add($c2); [void]$grid.Children.Add($c3)
    $w2.Content = $grid
    $w2.Show()
    $h2 = (New-Object Windows.Interop.WindowInteropHelper $w2).Handle
    [void][NativeBits]::SetWindowLong($h2, -20, ([NativeBits]::GetWindowLong($h2, -20) -bor 0x08000000 -bor 0x80 -bor 0x20))
    $S.fxw = $w2; $S.fxc = $c2; $S.lzc = $c3
    $win.Topmost = $false; $win.Topmost = $true       # keep him on top of the effects
}
function ClearItems {
    if ($null -ne $S.fxc) { $S.fxc.Children.Clear() }
    $S.balls.Clear(); $S.splats.Clear()
}
function PaintFail($ex) {
    $S.paintOk = $false; $S.inPaint = $false; $S.trap = $null; $S.chk = $null; $S.pathOn = $false; $S.map = $null
    $S.cf = $wa.Bottom; $S.cl = $wa.Left + 35; $S.cr = $wa.Right - 35; $S.cc = $wa.Top + 140; $S.sf = $wa.Bottom
    Write-Host ('StickBuddy: Paint features switched off after an error: ' + $ex.Exception.Message + ' (script line ' + $ex.InvocationInfo.ScriptLineNumber + ')') -ForegroundColor Yellow
    try { if ($null -ne $S.fxw) { $S.fxw.Visibility = 'Hidden' } } catch { }
}
function PaintEnter {
    $S.inPaint = $true
    EnsureFx
    $S.fxw.Visibility = 'Visible'
    $S.msg = 'ooh, paint!'; $S.msgT = $S.t + 2.2
    $S.nextItem = $S.t + 7
}
function PaintLeave {
    $S.inPaint = $false; $S.trap = $null; $S.chk = $null; $S.pathOn = $false; $S.map = $null; $S.cv = $null
    ClearItems
    ClearHoles
}
function PaintDetect($t) {
    if ($t -ge $S.prT) {                                   # look for a Paint window a few times a second
        $S.prT = $t + 0.4
        $f = [PaintBits]::Find()
        if ($null -ne $f) { $S.pwh = $f.H } else { $S.pwh = [IntPtr]::Zero }
    }
    $pw = [PaintBits]::RectOf($S.pwh)
    $m = $S.src.CompositionTarget.TransformFromDevice
    $was = $S.inPaint
    if ($null -ne $pw) {
        $nr = @{ L = $pw.L * $m.M11; T = $pw.T * $m.M22; R = $pw.R * $m.M11; B = $pw.B * $m.M22 }
        if ($was -and $null -ne $S.pr -and ($S.phase -in 'wander', 'lie', 'getup', 'chase')) {      # he rides along when you move the window
            $S.x += $nr.L - $S.pr.L; $S.y += $nr.T - $S.pr.T
        }
        $S.pr = $nr
        $S.pfg = [PaintBits]::IsForeground($pw.H)
        if (-not $was) { PaintEnter }
    } else {
        $S.pfg = $false
        if ($was) { PaintLeave }
        $S.pr = $null
    }
}
# floor / walls / ceiling for this frame: the screen, the Paint canvas, or the inside of a trap
function SetContext {
    $cf = $wa.Bottom; $cl = $wa.Left + 35; $cr = $wa.Right - 35; $cc = $wa.Top + 140
    if ($S.inPaint -and $null -ne $S.pr) {
        $cf = $S.pr.B - 40; $cl = $S.pr.L + 40; $cr = $S.pr.R - 40; $cc = $S.pr.T + 140
        if ($cc -gt ($cf - 130)) { $cc = $cf - 130 }
        if ($cl -gt $cr) { $cl = ($cl + $cr) / 2; $cr = $cl }
    }
    $S.sf = $cf
    if ($null -ne $S.trap) {
        $poly = $S.trap.poly
        $my = $S.y - 60
        $vs = PolySpan (PolyCuts $poly 'x' $S.x) $my
        if ($null -ne $vs) { $cf = $vs[1] - 2; $cc = $vs[0] + 128 }
        $hs = PolySpan (PolyCuts $poly 'y' $my) $S.x
        if ($null -ne $hs) { $cl = $hs[0] + 28; $cr = $hs[1] - 28 }
        if ($cc -gt $cf) { $cc = $cf }
        if ($cl -gt $cr) { $cl = ($cl + $cr) / 2; $cr = $cl }
    }
    $S.cf = $cf; $S.cl = $cl; $S.cr = $cr; $S.cc = $cc
}
# is there ink (anything not near-white) along these points? uses one screen grab; $null if the grab failed
function InkFlagsOf($poly) {
    $n = $poly.Count
    $xs = New-Object 'int[]' $n; $ys = New-Object 'int[]' $n
    $m = $S.src.CompositionTarget.TransformToDevice
    for ($i = 0; $i -lt $n; $i++) { $xs[$i] = [int]($poly[$i].X * $m.M11); $ys[$i] = [int]($poly[$i].Y * $m.M22) }
    $res = [PaintBits]::InkFlags($xs, $ys, 3)
    if ($null -eq $res) { return $null }
    , $res
}
# you let go of the mouse in Paint: was that a closed shape around him?
function AnalyzePath {
    $pts = $S.path
    if ($null -eq $pts -or $pts.Count -lt 10) { return }
    $minx = 100000.0; $miny = 100000.0; $maxx = -100000.0; $maxy = -100000.0; $len = 0.0
    for ($i = 0; $i -lt $pts.Count; $i++) {
        $q = $pts[$i]
        if ($q.X -lt $minx) { $minx = $q.X }
        if ($q.X -gt $maxx) { $maxx = $q.X }
        if ($q.Y -lt $miny) { $miny = $q.Y }
        if ($q.Y -gt $maxy) { $maxy = $q.Y }
        if ($i -gt 0) { $len += (Dist $q $pts[$i - 1]) }
    }
    if (($maxx - $minx) -lt 70 -or ($maxy - $miny) -lt 140) { return }       # too small to hold him
    $a = $pts[0]; $b = $pts[$pts.Count - 1]
    $cands = @()
    if ((Dist $a $b) -lt 34) {                                                # a freehand loop that comes back to where it began
        $cands += , (Resample $pts 12)
    } else {
        $dev = 0.0
        foreach ($q in $pts) { $dv = DistSeg $q $a $b; if ($dv -gt $dev) { $dev = $dv } }
        if ($dev -lt (0.12 * (Dist $a $b))) {                                 # a straight drag: the rectangle or ellipse tool
            $x0 = [Math]::Min($a.X, $b.X); $x1 = [Math]::Max($a.X, $b.X); $y0 = [Math]::Min($a.Y, $b.Y); $y1 = [Math]::Max($a.Y, $b.Y)
            if (($x1 - $x0) -ge 70 -and ($y1 - $y0) -ge 140) {
                $cands += , (Resample @((Pt $x0 $y0), (Pt $x1 $y0), (Pt $x1 $y1), (Pt $x0 $y1), (Pt $x0 $y0)) 12)
                $ell = @()
                for ($i = 0; $i -le 48; $i++) {
                    $th = $i * 2 * [Math]::PI / 48
                    $ell += (Pt (($x0 + $x1) / 2 + ($x1 - $x0) / 2 * [Math]::Cos($th)) (($y0 + $y1) / 2 + ($y1 - $y0) / 2 * [Math]::Sin($th)))
                }
                $cands += , (Resample $ell 12)
            }
        }
    }
    if ($cands.Count -gt 0) { $S.chk = @{ t = $S.t + 0.35; cands = $cands } }   # give Paint a moment to draw it, then check the ink
}
function GestureUpdate($cur, $lb, $t) {
    $inw = ($null -ne $S.pr) -and ($cur.X -ge $S.pr.L) -and ($cur.X -le $S.pr.R) -and ($cur.Y -ge $S.pr.T) -and ($cur.Y -le $S.pr.B)
    if ($lb -and $S.pfg -and $inw -and $S.phase -ne 'held') {
        if (-not $S.pathOn) {
            $S.pathOn = $true
            $S.path = New-Object System.Collections.ArrayList
            [void]$S.path.Add($cur)
        } elseif ((Dist $cur $S.path[$S.path.Count - 1]) -ge 4 -and $S.path.Count -lt 900) { [void]$S.path.Add($cur) }
    } elseif ($S.pathOn) {
        $S.pathOn = $false
        AnalyzePath
    }
}
function TrapUpdate($t) {
    if ($null -ne $S.chk -and $t -ge $S.chk.t) {
        $cands = $S.chk.cands; $S.chk = $null
        foreach ($cd in $cands) {
            if (-not (PolyInside $cd $S.x ($S.y - 60))) { continue }
            $flg = InkFlagsOf $cd
            if ($null -eq $flg) { continue }
            $cnt = 0
            foreach ($f in $flg) { if ($f) { $cnt += 1 } }
            if ($cnt -ge (0.88 * $cd.Count)) {                                  # the shape really is drawn on the canvas
                $S.trap = @{ poly = $cd; t0 = $t; next = $t + 1.0; lx = $S.x; ly = $S.y }
                $S.angry = $true; $S.msg = 'let me out!'; $S.msgT = $t + 2.5
                $S.lzFreeT = $t + 3.0                                           # (a few seconds of banging on the walls, then the lasers come out)
                if ($S.phase -in 'ride', 'takeoff', 'flyto', 'flyland', 'hover', 'crouch', 'hop', 'chase') { $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0 }
                break
            }
        }
    }
    if ($null -ne $S.trap) {
        if ($S.phase -ne 'held') {                                              # the walls are solid: if he ends up outside, put him back
            if (PolyInside $S.trap.poly $S.x ($S.y - 60)) { $S.trap.lx = $S.x; $S.trap.ly = $S.y }
            else { $S.x = $S.trap.lx; $S.y = $S.trap.ly; $S.vx = (0 - $S.vx) * 0.5; $S.vy = (0 - $S.vy) * 0.5 }
        }
        if ($t -ge $S.trap.next) {
            $S.trap.next = $t + 1.0
            $flg = InkFlagsOf $S.trap.poly
            if ($null -ne $flg) {
                $n = $flg.Count; $run = 0; $best = 0
                for ($i = 0; $i -lt (2 * $n); $i++) {                          # longest gap in the wall (it wraps around)
                    if ($flg[$i % $n]) { $run = 0 } else { $run += 1; if ($run -gt $best) { $best = $run } }
                }
                if ($best -ge 4 -or ($t - $S.trap.t0) -gt 150) {
                    $S.trap = $null; $S.msg = 'free!'; $S.msgT = $t + 1.5
                }
            }
        }
    }
}
function NewDot($r, $brush) {
    $e = New-Object Windows.Shapes.Ellipse
    $e.Width = 2 * $r; $e.Height = 2 * $r; $e.Fill = $brush; $e.IsHitTestVisible = $false
    [void]$S.fxc.Children.Add($e)
    $e
}
function SplatAt($x, $y, $r, $brush) {
    if ($S.splats.Count -gt 24) { $old = $S.splats[0]; $S.fxc.Children.Remove($old.el); $S.splats.RemoveAt(0) }
    $e = NewDot $r $brush
    [Windows.Controls.Canvas]::SetLeft($e, $x - $r - $vl); [Windows.Controls.Canvas]::SetTop($e, $y - $r - $vt)
    [void]$S.splats.Add(@{ x = $x; y = $y; r = $r; el = $e; hp = $r * 3.0; hp0 = $r * 3.0; die = $S.t + 14 })
}
# a paint ball (or a big bucket of paint) lobbed from his hand at your pointer
function SpawnBall($big) {
    $x0 = $S.x + 26; $y0 = $S.y - 104
    $dx = $S.cu.X - $x0; $dy = $S.cu.Y - $y0
    $dd = [Math]::Sqrt($dx * $dx + $dy * $dy)
    $ft = Clamp ($dd / 650) 0.45 1.1
    $r = 9; if ($big) { $r = 20 }
    $brush = $S.pal[$rng.Next($S.pal.Count)]
    $e = NewDot $r $brush
    [void]$S.balls.Add(@{ x = $x0; y = $y0; vx = $dx / $ft; vy = ($dy - 450 * $ft * $ft) / $ft; r = $r; br = $brush; el = $e; die = $S.t + 4 })
}
function Nudge($bl, $cur, $lb, $t) {
    if (-not $MouseNudge -or $lb -or -not $S.pfg -or ($t - $S.knockT) -lt 1.2) { return }
    $S.knockT = $t
    $l = [Math]::Max(1.0, [Math]::Sqrt($bl.vx * $bl.vx + $bl.vy * $bl.vy))
    $m = $S.src.CompositionTarget.TransformToDevice
    [void][PaintBits]::NudgeCursor([int](($cur.X + $bl.vx / $l * 45) * $m.M11), [int](($cur.Y + $bl.vy / $l * 45) * $m.M22))
}
function ItemsUpdate($t, $dt, $cur, $lb) {
    $mv = Dist $cur $S.pc
    for ($i = $S.balls.Count - 1; $i -ge 0; $i--) {
        $bl = $S.balls[$i]
        $bl.vy += 900 * $dt; $bl.x += $bl.vx * $dt; $bl.y += $bl.vy * $dt
        [Windows.Controls.Canvas]::SetLeft($bl.el, $bl.x - $bl.r - $vl); [Windows.Controls.Canvas]::SetTop($bl.el, $bl.y - $bl.r - $vt)
        $gone = $false
        if ((Dist (Pt $bl.x $bl.y) $cur) -lt ($bl.r + 12)) {
            SplatAt $cur.X $cur.Y ($bl.r * 2.6) $bl.br
            Nudge $bl $cur $lb $t
            $gone = $true
        } elseif (($bl.y -gt ($S.sf + 10)) -or ($t -gt $bl.die)) {
            SplatAt $bl.x ([Math]::Min($bl.y, $S.sf + 8)) ($bl.r * 1.6) $bl.br
            $gone = $true
        }
        if ($gone) { $S.fxc.Children.Remove($bl.el); $S.balls.RemoveAt($i) }
    }
    for ($i = $S.splats.Count - 1; $i -ge 0; $i--) {                           # splats fade, and you can scrub them away
        $sp = $S.splats[$i]
        if ($lb -and $mv -gt 0.5 -and (Dist (Pt $sp.x $sp.y) $cur) -lt $sp.r) { $sp.hp -= $mv * 1.5 }
        $life = [Math]::Min(1.0, ($sp.die - $t) / 2.0)
        $sp.el.Opacity = 0.85 * [Math]::Max(0.0, [Math]::Min($life, $sp.hp / $sp.hp0))
        if ($sp.hp -le 0 -or $t -ge $sp.die) { $S.fxc.Children.Remove($sp.el); $S.splats.RemoveAt($i) }
    }
}
# keeps a picture of the canvas so he knows where your lines are (refreshed a few times a second, only around him)
function MapUpdate($t) {
    $m = $S.src.CompositionTarget.TransformToDevice
    $S.dx = $m.M11; $S.dy = $m.M22
    if (-not $SolidDrawings -or -not $S.xOk -or -not $S.inPaint -or -not $S.pfg -or $null -eq $S.pr) { $S.map = $null; return }
    $key = [double]$S.pr.L * 7 + [double]$S.pr.T * 13 + [double]$S.pr.R * 17 + [double]$S.pr.B * 19
    if ($null -eq $S.cv -or $t -ge $S.cvT -or $key -ne $S.cvKey) {          # find the white canvas again every few seconds (and when you move the window)
        $S.cvT = $t + 2.5; $S.cvKey = $key
        $r0 = [int]($S.pr.L * $m.M11); $r1 = [int]($S.pr.T * $m.M22); $r2 = [int]($S.pr.R * $m.M11); $r3 = [int]($S.pr.B * $m.M22)
        $cvn = [XBits]::FindCanvas($r0, $r1, $r2 - $r0, $r3 - $r1)
        if ($null -ne $cvn) { $S.cv = $cvn }
    }
    if ($null -eq $S.cv) { $S.map = $null; return }
    $moved = Dist (Pt $S.x $S.y) $S.mapC
    if ($null -ne $S.map -and $t -lt $S.mapT -and $moved -lt 120) { return }
    $gap = 0.14; if ($S.pathOn) { $gap = 0.07 }
    $S.mapT = $t + $gap; $S.mapC = (Pt $S.x $S.y)
    $cx = [int]($S.x * $m.M11); $cy = [int](($S.y - 60) * $m.M22)
    $rw = [int](480 * $m.M11); $rh = [int](360 * $m.M22)
    $mp = [XBits]::Scan($cx - $rw, $cy - $rh, 2 * $rw, 2 * $rh, 4, $S.cv[0], $S.cv[1], $S.cv[2], $S.cv[3])
    if ($null -ne $mp) { $mp.Holes = $S.holeArr; $S.map = $mp }
}
function PaintUpdate($t, $dt, $cur, $lb) {
    PaintDetect $t
    if ($S.inPaint) {
        try { MapUpdate $t } catch {
            $S.xOk = $false; $S.map = $null
            Write-Host ('StickBuddy: solid drawings switched off after an error: ' + $_.Exception.Message) -ForegroundColor Yellow
        }
        GestureUpdate $cur $lb $t
        TrapUpdate $t
        if ($S.pfg -and $S.phase -ne 'held' -and $S.jt.Count -gt 0) { ScrubParts $cur $lb $dt }
        ItemsUpdate $t $dt $cur $lb
    }
    SetContext
}

# ---------- burn marks: places where his lasers burned your drawing away (he can walk and fly through them) ----------
function RefreshHoles {
    $lst = New-Object 'System.Collections.Generic.List[int]'
    foreach ($ho in $S.holes) { $lst.Add([int]($ho.x * $S.dx)); $lst.Add([int]($ho.y * $S.dy)); $lst.Add([int]($ho.r * $S.dx)) }
    $S.holeArr = $lst.ToArray()
    if ($null -ne $S.map) { $S.map.Holes = $S.holeArr }
}
function AddHole($x, $y, $r) {
    EnsureFx
    $e = New-Object Windows.Shapes.Ellipse
    $e.Width = 2 * $r; $e.Height = 2 * $r; $e.IsHitTestVisible = $false; $e.Fill = $S.scorch
    [Windows.Controls.Canvas]::SetLeft($e, $x - $r - $vl); [Windows.Controls.Canvas]::SetTop($e, $y - $r - $vt)
    [Windows.Controls.Panel]::SetZIndex($e, 1)
    [void]$S.lzc.Children.Add($e)
    [void]$S.holes.Add(@{ x = $x; y = $y; r = $r; die = $S.t + 18; el = $e })
    RefreshHoles
}
function ClearHoles {
    if ($null -ne $S.lzc) { foreach ($ho in $S.holes) { $S.lzc.Children.Remove($ho.el) } }
    $S.holes.Clear()
    RefreshHoles
}
function HoleTick($t) {
    if ($S.holes.Count -eq 0) { return }
    $dirty = $false
    for ($i = $S.holes.Count - 1; $i -ge 0; $i--) {
        $ho = $S.holes[$i]
        $life = $ho.die - $t
        if ($life -le 0) { $S.lzc.Children.Remove($ho.el); $S.holes.RemoveAt($i); $dirty = $true }
        else {
            $glow = 1.0
            if ((18 - $life) -lt 2.0) { $glow = 0.8 + 0.2 * [Math]::Sin($t * 22) }       # still hot
            $ho.el.Opacity = [Math]::Min(1.0, $life / 3.0) * $glow
        }
    }
    if ($dirty) { RefreshHoles }
}
# a laser has been burning here long enough: a hole appears (and if it is in his cage wall, he is free)
function BurnAt($x, $y) {
    AddHole $x $y 30
    if ($null -ne $S.trap) {
        $here = Pt $x $y
        foreach ($q in $S.trap.poly) {
            if ((Dist $q $here) -lt 46) { $S.trap = $null; $S.msg = 'free!'; $S.msgT = $S.t + 1.5; break }
        }
    }
}

# ---------- lasers ----------
function EnsureLaser {
    EnsureFx
    if ($null -ne $S.lzE) { return }
    $le = @{ lines = @(); fl = @() }
    for ($k = 0; $k -lt 2; $k++) {
        foreach ($spec in @( @(18.0, 70, 255, 40, 30), @(8.0, 190, 255, 70, 50), @(2.8, 255, 255, 255, 255) )) {
            $ln = New-Object Windows.Shapes.Line
            $ln.Stroke = (BrushA $spec[1] $spec[2] $spec[3] $spec[4])
            $ln.StrokeThickness = $spec[0]; $ln.Tag = $spec[0]
            $ln.StrokeStartLineCap = 'Round'; $ln.StrokeEndLineCap = 'Round'
            $ln.IsHitTestVisible = $false; $ln.Visibility = 'Hidden'
            [Windows.Controls.Panel]::SetZIndex($ln, 10)
            [void]$S.lzc.Children.Add($ln)
            $le.lines += $ln
        }
    }
    for ($k = 0; $k -lt 4; $k++) {                         # two glows where the beams leave his eyes, two where they hit
        $f = New-Object Windows.Shapes.Ellipse
        $f.IsHitTestVisible = $false; $f.Visibility = 'Hidden'
        if ($k -lt 2) { $f.Fill = (NewFlareBrush 1) } else { $f.Fill = (NewFlareBrush 2) }
        [Windows.Controls.Panel]::SetZIndex($f, 11)
        [void]$S.lzc.Children.Add($f)
        $le.fl += $f
    }
    $S.lzE = $le
}
function LaserUpdate($t, $dt) {
    $want = 0.0; if ($S.lzOn) { $want = 1.0 }
    $S.lzPow += ($want - $S.lzPow) * (1 - [Math]::Exp(-$dt * 14))
    if ($null -eq $S.lzE) { return }
    $le = $S.lzE
    if ($S.lzPow -lt 0.03) {
        foreach ($ob in $le.lines) { $ob.Visibility = 'Hidden' }
        foreach ($ob in $le.fl) { $ob.Visibility = 'Hidden' }
        $S.burnT = 0.0
        return
    }
    $pow = $S.lzPow * (0.9 + 0.1 * [Math]::Sin($t * 70))
    $face = $S.face
    $eyes = @( (ToScr (Pt ($S.head.X + $face * 17) ($S.head.Y - 4))), (ToScr (Pt ($S.head.X + $face * 17) ($S.head.Y + 4))) )
    $hitAny = $false
    for ($k = 0; $k -lt 2; $k++) {
        $ex = $eyes[$k].X; $ey = $eyes[$k].Y
        $ang0 = [Math]::Atan2($S.aimY - $ey, $S.aimX - $ex) + ($k * 2 - 1) * 0.045      # the two beams fan out a little
        $len = [Math]::Max(30.0, [Math]::Sqrt(($S.aimX - $ex) * ($S.aimX - $ex) + ($S.aimY - $ey) * ($S.aimY - $ey)))
        $ux = [Math]::Cos($ang0); $uy = [Math]::Sin($ang0)
        $reach = $len
        if ($null -ne $S.map) {
            $hd = $S.map.RayHit([int]($ex * $S.dx), [int]($ey * $S.dy), [int](($ex + $ux * $len) * $S.dx), [int](($ey + $uy * $len) * $S.dy), 3)
            if ($hd -ge 0) { $reach = $hd / $S.dx; $hitAny = $true; $S.lzHx = $ex + $ux * $reach; $S.lzHy = $ey + $uy * $reach }
        }
        $sx0 = $ex + $ux * 8; $sy0 = $ey + $uy * 8                  # (the beam starts just outside his head)
        $x2 = $ex + $ux * $reach; $y2 = $ey + $uy * $reach
        for ($q = 0; $q -lt 3; $q++) {
            $ln = $le.lines[$k * 3 + $q]
            $ln.X1 = $sx0 - $vl; $ln.Y1 = $sy0 - $vt; $ln.X2 = $x2 - $vl; $ln.Y2 = $y2 - $vt
            $ln.StrokeThickness = [double]$ln.Tag * $pow
            $ln.Visibility = 'Visible'
        }
        $fe = $le.fl[$k]; $sz = 24 * $pow
        $fe.Width = $sz; $fe.Height = $sz
        [Windows.Controls.Canvas]::SetLeft($fe, $sx0 - $sz / 2 - $vl); [Windows.Controls.Canvas]::SetTop($fe, $sy0 - $sz / 2 - $vt)
        $fe.Visibility = 'Visible'
        $fh = $le.fl[2 + $k]; $sz2 = (34 + 12 * [Math]::Sin($t * 45 + $k * 2)) * $pow
        $fh.Width = $sz2; $fh.Height = $sz2
        [Windows.Controls.Canvas]::SetLeft($fh, $x2 - $sz2 / 2 - $vl); [Windows.Controls.Canvas]::SetTop($fh, $y2 - $sz2 / 2 - $vt)
        $fh.Visibility = 'Visible'
    }
    if ($hitAny) { $S.burnT += $dt } else { $S.burnT = [Math]::Max(0.0, $S.burnT - $dt * 2) }
    if ($S.burnT -gt 0.7) { $S.burnT = 0.0; BurnAt $S.lzHx $S.lzHy }          # the beam has been on a line long enough: it burns through
}
# angry flight arrives: he hovers in front of you and fires, then (maybe) zaps your desktop back to 2001, or slices your window
function StartHover($t) {
    $S.phase = 'hover'; $S.hz = 'lock'; $S.hzt = $t
    $S.aimX = $S.x; $S.aimY = $S.y - 80                    # (the beams swing round to your pointer)
    $S.lzOn = $true
    $S.doSlice = $false; $S.doRetro = $false
    if ($S.rtOk -and $S.rt.state -eq 'off' -and ($rng.NextDouble() -lt $RetroChance)) { $S.doRetro = $true }
    elseif ($WindowBreak -and $S.xOk -and ($rng.NextDouble() -lt $BreakChance)) { $S.doSlice = $true }
    EnsureLaser
}
function EndHover($t, $cur) {
    $S.lzOn = $false
    $S.phase = 'ride'; $S.angry = $false; $S.rideT = $t; $S.pt = $t; $S.still = $t; $S.lc = $cur
}
# trapped and angry: pick a spot in the wall and burn through it
function StartBurn($t, $cur) {
    $best = $null; $bd = 1000000.0
    foreach ($q in $S.trap.poly) {
        $side = ($q.Y -lt ($S.y - 20)) -and ($q.Y -gt ($S.y - 120))        # a part of the wall at about his height
        if (-not $S.canFly -and -not $side) { continue }
        $d = Dist $q $cur
        if ($side) { $d = $d * 0.7 }
        if ($d -lt $bd) { $bd = $d; $best = $q }
    }
    if ($null -eq $best) { $S.lzFreeT = $t + 3.0; return }
    $S.bp = $best
    $S.phase = 'burn'; $S.pt = $t
    $S.aimX = $S.x; $S.aimY = $S.y - 70
    $S.lzOn = $true
    EnsureLaser
}

# ---------- slicing your front window in half (an illusion: a snapshot of it, cut in two, falls away) ----------
function SliceStart($t) {
    if (-not $S.xOk -or $null -ne $S.sl) { return $false }
    try {
        $rc = [XBits]::ForeRect()
        if ($null -eq $rc) { return $false }
        EnsureFx; EnsureLaser
        $m = $S.src.CompositionTarget.TransformFromDevice
        $pw = $rc[2] - $rc[0]; $ph = $rc[3] - $rc[1]
        $bmp = [System.Drawing.Bitmap]::new($pw, $ph, [System.Drawing.Imaging.PixelFormat]::Format32bppRgb)
        $gfx = [System.Drawing.Graphics]::FromImage($bmp)
        $gfx.CopyFromScreen($rc[0], $rc[1], 0, 0, [System.Drawing.Size]::new($pw, $ph))
        $gfx.Dispose()
        $hb = $bmp.GetHbitmap()
        $src = [System.Windows.Interop.Imaging]::CreateBitmapSourceFromHBitmap($hb, [IntPtr]::Zero, [System.Windows.Int32Rect]::Empty, [System.Windows.Media.Imaging.BitmapSizeOptions]::FromEmptyOptions())
        [void][XBits]::DeleteObject($hb)
        $bmp.Dispose()
        $src.Freeze()
        $wl = $rc[0] * $m.M11; $wtop = $rc[1] * $m.M22          # where the window is (in screen units) and how big
        $ww = $pw * $m.M11; $wh = $ph * $m.M22
        if ($rng.NextDouble() -lt 0.5) {                        # a cut from the top edge to the bottom edge...
            $ax = $ww * (0.3 + 0.4 * $rng.NextDouble()); $ay = 0.0
            $bx = $ax + $ww * (0.36 * $rng.NextDouble() - 0.18); $by = $wh
        } else {                                                # ...or from the left edge to the right edge
            $ay = $wh * (0.3 + 0.4 * $rng.NextDouble()); $ax = 0.0
            $by = $ay + $wh * (0.36 * $rng.NextDouble() - 0.18); $bx = $ww
        }
        $bx = Clamp $bx 0.0 $ww; $by = Clamp $by 0.0 $wh
        $clen = [Math]::Max(1.0, [Math]::Sqrt(($bx - $ax) * ($bx - $ax) + ($by - $ay) * ($by - $ay)))
        $ux = ($bx - $ax) / $clen; $uy = ($by - $ay) / $clen
        $nx = $uy; $ny = 0 - $ux
        $rectPts = @( (Pt 0 0), (Pt $ww 0), (Pt $ww $wh), (Pt 0 $wh) )
        $polyA = ClipHalf $rectPts $ax $ay $nx $ny 1
        $polyB = ClipHalf $rectPts $ax $ay $nx $ny (-1)
        if ($polyA.Count -lt 3 -or $polyB.Count -lt 3) { return $false }
        $void = New-Object Windows.Shapes.Rectangle
        $void.Width = $ww; $void.Height = $wh; $void.Fill = (Brush 4 4 8); $void.Opacity = 0.0; $void.IsHitTestVisible = $false
        [Windows.Controls.Canvas]::SetLeft($void, $wl - $vl); [Windows.Controls.Canvas]::SetTop($void, $wtop - $vt)
        [Windows.Controls.Panel]::SetZIndex($void, 4)
        [void]$S.lzc.Children.Add($void)
        $imgs = @(); $rots = @(); $trs = @()
        foreach ($pair in @( @($polyA, (Centroid $polyA)), @($polyB, (Centroid $polyB)) )) {
            $im = New-Object Windows.Controls.Image
            $im.Source = $src; $im.Width = $ww; $im.Height = $wh; $im.Stretch = 'Fill'; $im.IsHitTestVisible = $false
            $im.Clip = (PolyGeo $pair[0])
            $rt = New-Object Windows.Media.RotateTransform
            $rt.CenterX = $pair[1].X; $rt.CenterY = $pair[1].Y
            $tr = New-Object Windows.Media.TranslateTransform
            $tg = New-Object Windows.Media.TransformGroup
            [void]$tg.Children.Add($rt); [void]$tg.Children.Add($tr)
            $im.RenderTransform = $tg
            [Windows.Controls.Canvas]::SetLeft($im, $wl - $vl); [Windows.Controls.Canvas]::SetTop($im, $wtop - $vt)
            [Windows.Controls.Panel]::SetZIndex($im, 6)
            [void]$S.lzc.Children.Add($im)
            $imgs += $im; $rots += $rt; $trs += $tr
        }
        $cuts = @()                                             # the glowing cut line, drawn as his lasers sweep across
        foreach ($spec in @( @(14.0, 90, 255, 150, 40), @(6.0, 220, 255, 210, 90), @(2.2, 255, 255, 255, 255) )) {
            $cl2 = New-Object Windows.Shapes.Line
            $cl2.Stroke = (BrushA $spec[1] $spec[2] $spec[3] $spec[4]); $cl2.StrokeThickness = $spec[0]
            $cl2.StrokeStartLineCap = 'Round'; $cl2.StrokeEndLineCap = 'Round'; $cl2.IsHitTestVisible = $false
            $cl2.X1 = $wl - $vl + $ax; $cl2.Y1 = $wtop - $vt + $ay; $cl2.X2 = $cl2.X1; $cl2.Y2 = $cl2.Y1
            [Windows.Controls.Panel]::SetZIndex($cl2, 8)
            [void]$S.lzc.Children.Add($cl2)
            $cuts += $cl2
        }
        $S.sl = @{ state = 'sweep'; t0 = $t; void = $void; imgs = $imgs; rots = $rots; trs = $trs; cuts = $cuts
                   wl = $wl; wt = $wtop; ww = $ww; wh = $wh; ax = $ax; ay = $ay; bx = $bx; by = $by; nx = $nx; ny = $ny }
        $S.aimX = $wl + $ax; $S.aimY = $wtop + $ay
        return $true
    } catch {
        if (-not $S.lzErr) { $S.lzErr = $true; Write-Host ('StickBuddy: window slicing failed: ' + $_.Exception.Message) -ForegroundColor Yellow }
        try { SliceEnd } catch { }
        return $false
    }
}
function SliceSweep($prog) {                                   # the lasers draw the cut line from one edge to the other
    $sl = $S.sl
    if ($null -eq $sl) { return }
    $ez = $prog * $prog * (3 - 2 * $prog)
    $cx2 = $sl.ax + ($sl.bx - $sl.ax) * $ez; $cy2 = $sl.ay + ($sl.by - $sl.ay) * $ez
    foreach ($cl2 in $sl.cuts) { $cl2.X2 = $sl.wl - $vl + $cx2; $cl2.Y2 = $sl.wt - $vt + $cy2 }
    $S.aimX = $sl.wl + $cx2; $S.aimY = $sl.wt + $cy2
}
function SliceShatter($t) {                                    # snap! the two halves come apart
    $sl = $S.sl
    if ($null -eq $sl -or $sl.state -ne 'sweep') { return }
    $sl.state = 'fall'; $sl.t0 = $t
    $sl.void.Opacity = 1.0
}
function SliceEnd {
    $sl = $S.sl
    if ($null -eq $sl) { return }
    foreach ($im in $sl.imgs) { $im.Source = $null; $S.lzc.Children.Remove($im) }
    foreach ($cl2 in $sl.cuts) { $S.lzc.Children.Remove($cl2) }
    $S.lzc.Children.Remove($sl.void)
    $S.sl = $null
}
function SliceUpdate($t, $dt) {
    $sl = $S.sl
    if ($null -eq $sl) { return }
    if ($sl.state -eq 'sweep') {
        foreach ($cl2 in $sl.cuts) { $cl2.Opacity = 0.75 + 0.25 * [Math]::Sin($t * 40) }
        return
    }
    $tau = $t - $sl.t0
    $sep = 0.5 * 240 * $tau * $tau; $drop = 0.5 * 1100 * $tau * $tau
    for ($i = 0; $i -lt 2; $i++) {
        $sd = 1 - 2 * $i                                       # one half goes one way, the other half the other way
        $sl.trs[$i].X = $sd * $sl.nx * $sep
        $sl.trs[$i].Y = $sd * $sl.ny * $sep + $drop
        $sl.rots[$i].Angle = $sd * 16 * $tau * $tau
    }
    $fade = 1.0
    if ($tau -gt 2.2) { $fade = [Math]::Max(0.0, 1 - ($tau - 2.2) / 0.5) }       # then it all comes back
    foreach ($im in $sl.imgs) { $im.Opacity = $fade }
    $sl.void.Opacity = $fade
    $glow = 0.0; if ($tau -lt 0.6) { $glow = 1.0 - $tau / 0.6 }
    foreach ($cl2 in $sl.cuts) { $cl2.Opacity = $glow }
    if ($tau -gt 2.8) { SliceEnd }
}

# ---------- the 2001 zap: your desktop turns into a Windows XP look-alike for a while, then he changes it back ----------
# What really happens: (1) a gentle white flash, (2) your wallpaper is swapped for a Bliss-style hill (this session only,
# never saved), (3) a blue taskbar with a green Start button is drawn over your real taskbar (click-through: your real one
# keeps working underneath) and a Start menu pops open for a few seconds, (4) after $RetroSeconds another flash and it all goes back.
# Your windows themselves are never touched.

# small builders: a rectangle and a piece of text on a canvas
function RtBox($parent, $x, $y, $bw, $bh, $fill, $rad) {
    $e = New-Object Windows.Shapes.Rectangle
    $e.Width = $bw; $e.Height = $bh; $e.Fill = $fill; $e.IsHitTestVisible = $false
    if ($rad -gt 0) { $e.RadiusX = $rad; $e.RadiusY = $rad }
    [Windows.Controls.Canvas]::SetLeft($e, $x); [Windows.Controls.Canvas]::SetTop($e, $y)
    [void]$parent.Children.Add($e)
}
function RtText($parent, $txt, $x, $y, $size, $brush, $bold, $family) {
    $e = New-Object Windows.Controls.TextBlock
    $e.Text = $txt; $e.FontSize = $size; $e.Foreground = $brush; $e.IsHitTestVisible = $false
    $e.FontFamily = [Windows.Media.FontFamily]::new($family)
    if ($bold) { $e.FontWeight = 'Bold' }
    [Windows.Controls.Canvas]::SetLeft($e, $x); [Windows.Controls.Canvas]::SetTop($e, $y)
    [void]$parent.Children.Add($e)
}

# a hill-and-sky picture in the style of the old default wallpaper, drawn with shapes (so there is no image to ship)
function MakeBliss($pw, $ph) {
    $cv = New-Object Windows.Controls.Canvas
    $cv.Width = $pw; $cv.Height = $ph
    $sky = New-Object Windows.Media.LinearGradientBrush
    $sky.StartPoint = [Windows.Point]::new(0, 0); $sky.EndPoint = [Windows.Point]::new(0, 1)
    foreach ($gs in @( @(0.0, 255, 22, 84, 200), @(0.4, 255, 52, 130, 226), @(0.7, 255, 118, 183, 244), @(1.0, 255, 186, 222, 248) )) {
        [void]$sky.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb($gs[1], $gs[2], $gs[3], $gs[4]), $gs[0]))
    }
    $cv.Background = $sky
    # soft white clouds: x, y, width, height (as fractions of the picture) and how solid
    foreach ($cd in @( @(0.02, 0.04, 0.36, 0.20, 0.95), @(0.18, 0.10, 0.28, 0.14, 0.85), @(0.55, 0.02, 0.45, 0.24, 0.90), @(0.70, 0.12, 0.30, 0.14, 0.80),
                       @(0.32, 0.24, 0.26, 0.10, 0.70), @(0.62, 0.30, 0.28, 0.10, 0.75), @(0.00, 0.28, 0.20, 0.08, 0.60), @(0.80, 0.26, 0.20, 0.08, 0.70), @(0.44, 0.05, 0.18, 0.08, 0.70) )) {
        $e = New-Object Windows.Shapes.Ellipse
        $e.Width = $cd[2] * $pw; $e.Height = $cd[3] * $ph
        $rb = New-Object Windows.Media.RadialGradientBrush
        [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb([int](255 * $cd[4]), 255, 255, 255), 0.0))
        [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb([int](150 * $cd[4]), 255, 255, 255), 0.5))
        [void]$rb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb(0, 255, 255, 255), 1.0))
        $e.Fill = $rb
        [Windows.Controls.Canvas]::SetLeft($e, $cd[0] * $pw); [Windows.Controls.Canvas]::SetTop($e, $cd[1] * $ph)
        [void]$cv.Children.Add($e)
    }
    # the green hill
    $hg = New-Object Windows.Media.StreamGeometry
    $gc = $hg.Open()
    $gc.BeginFigure([Windows.Point]::new(0, 0.60 * $ph), $true, $true)
    $gc.BezierTo([Windows.Point]::new(0.30 * $pw, 0.40 * $ph), [Windows.Point]::new(0.62 * $pw, 0.47 * $ph), [Windows.Point]::new($pw, 0.64 * $ph), $true, $true)
    $gc.LineTo([Windows.Point]::new($pw, $ph), $true, $true)
    $gc.LineTo([Windows.Point]::new(0, $ph), $true, $true)
    $gc.Close()
    $hp = New-Object Windows.Shapes.Path
    $hp.Data = $hg
    $hp.Fill = (New-Grad @( @(0.0, 255, 158, 210, 62), @(0.18, 255, 112, 178, 40), @(0.55, 255, 62, 140, 28), @(1.0, 255, 36, 100, 20) ))
    [void]$cv.Children.Add($hp)
    # a sunny patch on the hill
    $sun = New-Object Windows.Shapes.Ellipse
    $sun.Width = 0.7 * $pw; $sun.Height = 0.25 * $ph
    $sb = New-Object Windows.Media.RadialGradientBrush
    [void]$sb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb(120, 214, 240, 110), 0.0))
    [void]$sb.GradientStops.Add([Windows.Media.GradientStop]::new([Windows.Media.Color]::FromArgb(0, 214, 240, 110), 1.0))
    $sun.Fill = $sb
    [Windows.Controls.Canvas]::SetLeft($sun, 0.12 * $pw); [Windows.Controls.Canvas]::SetTop($sun, 0.46 * $ph)
    [void]$cv.Children.Add($sun)
    $cv.Measure([Windows.Size]::new($pw, $ph))
    $cv.Arrange([Windows.Rect]::new(0, 0, $pw, $ph))
    $cv.UpdateLayout()
    $rtb = [Windows.Media.Imaging.RenderTargetBitmap]::new($pw, $ph, 96, 96, [Windows.Media.PixelFormats]::Pbgra32)
    $rtb.Render($cv)
    return $rtb
}
function SaveWallpaper($pw, $ph) {
    $bmp = MakeBliss $pw $ph
    $path = Join-Path ([IO.Path]::GetTempPath()) 'stickbuddy_xp_wallpaper.png'
    $enc = New-Object Windows.Media.Imaging.PngBitmapEncoder
    $enc.Frames.Add([Windows.Media.Imaging.BitmapFrame]::Create($bmp))
    $fs = [IO.File]::Create($path)
    try { $enc.Save($fs) } finally { $fs.Dispose() }
    return $path
}
# swaps your wallpaper for the hill ($true) or puts yours back ($false). Never written to your settings.
function RetroWall($on) {
    if (-not $RetroWallpaper) { return }
    $r = $S.rt
    if ($on) {
        if (-not $r.saved) {
            $orig = $null
            try { $orig = (Get-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name WallPaper -ErrorAction Stop).WallPaper } catch { }
            if ([string]::IsNullOrEmpty($orig) -or -not (Test-Path -LiteralPath $orig)) { $orig = [RetroBits]::GetWallpaper() }
            if ($null -eq $orig) { return }                    # can't tell what your wallpaper is, so it is left alone
            $r.orig = $orig; $r.saved = $true
        }
        $scr = [Windows.Forms.Screen]::PrimaryScreen.Bounds
        $r.file = SaveWallpaper $scr.Width $scr.Height
        if ([RetroBits]::SetWallpaper($r.file)) { $r.hasWall = $true }
    } elseif ($r.hasWall) {
        [void][RetroBits]::SetWallpaper($r.orig)
        $r.hasWall = $false; $r.saved = $false
        try { if (Test-Path -LiteralPath $r.file) { Remove-Item -LiteralPath $r.file -Force } } catch { }
    }
}

# the blue taskbar with the green Start button and the clock; null if your real taskbar is not along the bottom of the main screen
function RetroMakeTaskbar {
    $w2 = [Windows.SystemParameters]::WorkArea
    $pw = [Windows.SystemParameters]::PrimaryScreenWidth; $ph = [Windows.SystemParameters]::PrimaryScreenHeight
    $th = $ph - $w2.Bottom
    if ($th -lt 24 -or $w2.Left -ne 0 -or $w2.Top -ne 0 -or [Math]::Abs($w2.Width - $pw) -gt 1) { return $null }
    $sc = Clamp ($th / 30.0) 1.0 1.8
    $cv = New-Object Windows.Controls.Canvas
    $cv.Width = $pw; $cv.Height = $th; $cv.IsHitTestVisible = $false
    [Windows.Controls.Canvas]::SetLeft($cv, 0 - $vl); [Windows.Controls.Canvas]::SetTop($cv, $w2.Bottom - $vt)
    [Windows.Controls.Panel]::SetZIndex($cv, 2)
    RtBox $cv 0 0 $pw $th (New-Grad @( @(0.0, 255, 66, 128, 236), @(0.08, 255, 41, 100, 221), @(0.5, 255, 36, 94, 220), @(0.9, 255, 28, 78, 186), @(1.0, 255, 22, 64, 154) )) 0
    # the tray (right) with the clock
    $trayW = 100 * $sc
    RtBox $cv ($pw - $trayW) 0 $trayW $th (New-Grad @( @(0.0, 255, 24, 150, 236), @(0.1, 255, 15, 138, 229), @(0.9, 255, 11, 112, 204), @(1.0, 255, 9, 96, 184) )) 0
    RtBox $cv ($pw - $trayW) 0 1 $th (Brush 16 66 178) 0
    $dots = @( (Brush 200 200 210), (Brush 90 200 90), (Brush 230 80 70) )
    for ($i = 0; $i -lt 3; $i++) { RtBox $cv ($pw - $trayW + 8 * $sc + $i * 12 * $sc) (($th - 9 * $sc) / 2) (9 * $sc) (9 * $sc) $dots[$i] (4 * $sc) }
    $clk = New-Object Windows.Controls.TextBlock
    $clk.Text = (Get-Date).ToString('h:mm tt'); $clk.FontSize = 12 * $sc; $clk.Foreground = $white; $clk.IsHitTestVisible = $false
    $clk.FontFamily = [Windows.Media.FontFamily]::new('Tahoma')
    [Windows.Controls.Canvas]::SetLeft($clk, $pw - $trayW + 46 * $sc); [Windows.Controls.Canvas]::SetTop($clk, ($th - 16 * $sc) / 2)
    [void]$cv.Children.Add($clk)
    # the green Start button
    $bw = 118 * $sc
    $sb = New-Object Windows.Controls.Border
    $sb.Width = $bw; $sb.Height = $th; $sb.IsHitTestVisible = $false
    $sb.CornerRadius = [Windows.CornerRadius]::new(0, 10 * $sc, 10 * $sc, 0)
    $sb.Background = (New-Grad @( @(0.0, 255, 118, 200, 98), @(0.12, 255, 70, 170, 62), @(0.5, 255, 56, 150, 50), @(0.85, 255, 40, 122, 36), @(1.0, 255, 34, 104, 30) ))
    [void]$cv.Children.Add($sb)
    $lx = 14 * $sc; $ly = ($th - 18 * $sc) / 2
    RtBox $cv $lx $ly (8 * $sc) (8 * $sc) (Brush 242 84 44) (1 * $sc)
    RtBox $cv ($lx + 10 * $sc) $ly (8 * $sc) (8 * $sc) (Brush 120 196 48) (1 * $sc)
    RtBox $cv $lx ($ly + 10 * $sc) (8 * $sc) (8 * $sc) (Brush 52 136 236) (1 * $sc)
    RtBox $cv ($lx + 10 * $sc) ($ly + 10 * $sc) (8 * $sc) (8 * $sc) (Brush 250 202 44) (1 * $sc)
    $st = New-Object Windows.Controls.TextBlock
    $st.Text = 'start'; $st.FontSize = 20 * $sc; $st.Foreground = $white; $st.IsHitTestVisible = $false
    $st.FontFamily = [Windows.Media.FontFamily]::new('Trebuchet MS'); $st.FontStyle = 'Italic'; $st.FontWeight = 'Bold'
    [Windows.Controls.Canvas]::SetLeft($st, 40 * $sc); [Windows.Controls.Canvas]::SetTop($st, ($th - 26 * $sc) / 2)
    [void]$cv.Children.Add($st)
    return @{ cv = $cv; clk = $clk; w = $pw; h = $th; s = $sc; bw = $bw; trayW = $trayW; top = $w2.Bottom }
}
# one taskbar button per open window (refreshed every few seconds)
function RetroButtons {
    $r = $S.rt; $ti = $r.tbi
    if ($null -eq $ti) { return }
    foreach ($b0 in $r.btns) { $ti.cv.Children.Remove($b0) }
    $r.btns.Clear()
    $titles = [RetroBits]::Titles(8)
    $fore = [RetroBits]::ForeTitle()
    $n = $titles.Count
    if ($n -eq 0) { return }
    $x0 = $ti.bw + 10 * $ti.s
    $avail = $ti.w - $x0 - $ti.trayW - 10
    $bw2 = [Math]::Min(170 * $ti.s, $avail / $n - 4)
    if ($bw2 -lt 40) { return }
    for ($i = 0; $i -lt $n; $i++) {
        $act = ($titles[$i] -eq $fore)
        $bd = New-Object Windows.Controls.Border
        $bd.Width = $bw2; $bd.Height = $ti.h - 6 * $ti.s; $bd.IsHitTestVisible = $false
        $bd.CornerRadius = [Windows.CornerRadius]::new(3 * $ti.s)
        $bd.BorderThickness = [Windows.Thickness]::new(1)
        if ($act) {
            $bd.Background = (New-Grad @( @(0.0, 255, 30, 82, 183), @(1.0, 255, 40, 100, 212) )); $bd.BorderBrush = (Brush 14 40 124)
        } else {
            $bd.Background = (New-Grad @( @(0.0, 255, 92, 150, 246), @(0.5, 255, 60, 124, 238), @(1.0, 255, 44, 104, 224) )); $bd.BorderBrush = (Brush 18 62 170)
        }
        $tx = New-Object Windows.Controls.TextBlock
        $tx.Text = $titles[$i]; $tx.Foreground = $white; $tx.FontFamily = [Windows.Media.FontFamily]::new('Tahoma'); $tx.FontSize = 11 * $ti.s
        $tx.TextTrimming = 'CharacterEllipsis'; $tx.Margin = [Windows.Thickness]::new(8 * $ti.s, 0, 4, 0); $tx.VerticalAlignment = 'Center'
        $bd.Child = $tx
        [Windows.Controls.Canvas]::SetLeft($bd, $x0 + $i * ($bw2 + 4)); [Windows.Controls.Canvas]::SetTop($bd, 3 * $ti.s)
        [void]$ti.cv.Children.Add($bd)
        [void]$r.btns.Add($bd)
    }
}
# the open Start menu (the look from the screenshots), 380 x 470
function RetroMakeMenu {
    $m = New-Object Windows.Controls.Canvas
    $m.Width = 380; $m.Height = 470; $m.IsHitTestVisible = $false
    $body = New-Object Windows.Shapes.Rectangle
    $body.Width = 380; $body.Height = 470; $body.Fill = (Brush 255 255 255); $body.Stroke = (Brush 22 72 196); $body.StrokeThickness = 2; $body.IsHitTestVisible = $false
    [void]$m.Children.Add($body)
    $uname = $env:USERNAME
    if ([string]::IsNullOrEmpty($uname)) { $uname = 'user' }
    RtBox $m 1 1 378 58 (New-Grad @( @(0.0, 255, 31, 86, 214), @(0.5, 255, 56, 120, 235), @(1.0, 255, 36, 92, 218) )) 0
    RtBox $m 10 6 46 46 (New-Grad @( @(0.0, 255, 235, 150, 60), @(1.0, 255, 190, 80, 30) )) 5
    RtBox $m 24 12 18 18 (Brush 255 240 220) 9
    RtBox $m 17 32 32 18 (Brush 255 240 220) 9
    RtText $m $uname 66 16 20 $white $true 'Trebuchet MS'
    RtBox $m 1 59 189 366 (Brush 255 255 255) 0
    RtBox $m 190 59 189 366 (Brush 211 229 250) 0
    RtBox $m 190 59 1 366 (Brush 153 186 229) 0
    RtBox $m 1 425 378 44 (New-Grad @( @(0.0, 255, 36, 92, 218), @(1.0, 255, 31, 80, 200) )) 0
    $ic = @( (Brush 60 130 230), (Brush 240 150 40), (Brush 80 180 90), (Brush 150 90 200), (Brush 230 190 50), (Brush 200 70 70) )
    # left column: the two big pinned programs, then the others
    RtBox $m 10 66 32 32 $ic[0] 5
    RtText $m 'Internet' 50 67 13 (Brush 0 0 0) $true 'Tahoma'
    RtText $m 'Internet' 50 84 11 (Brush 120 120 120) $false 'Tahoma'
    RtBox $m 10 108 32 32 $ic[1] 5
    RtText $m 'E-mail' 50 109 13 (Brush 0 0 0) $true 'Tahoma'
    RtText $m 'E-mail' 50 126 11 (Brush 120 120 120) $false 'Tahoma'
    RtBox $m 10 148 170 1 (Brush 210 210 210) 0
    $small = @('Notepad', 'Calculator', 'Paint', 'Command Prompt', 'Music Player', 'Photo Viewer')
    $yy = 158; $k = 0
    foreach ($nm in $small) {
        $ci = $ic[$k % 6]
        RtBox $m 12 $yy 24 24 $ci 4
        RtText $m $nm 46 ($yy + 4) 12 (Brush 0 0 0) $false 'Tahoma'
        $yy += 36; $k++
    }
    RtBox $m 10 392 170 1 (Brush 210 210 210) 0
    RtText $m 'All Programs' 56 402 12 (Brush 0 0 0) $true 'Tahoma'
    RtBox $m 150 398 24 24 (Brush 60 170 60) 12
    # right column
    $rn = @('My Documents', 'My Recent Documents', 'My Pictures', 'My Music', 'My Computer')
    $yy = 68; $k = 0
    foreach ($nm in $rn) {
        $ci = $ic[($k + 2) % 6]
        RtBox $m 198 $yy 26 26 $ci 4
        RtText $m $nm 232 ($yy + 5) 12 (Brush 0 40 110) $true 'Tahoma'
        $yy += 33; $k++
    }
    RtBox $m 198 236 170 1 (Brush 160 190 230) 0
    $rn2 = @('Control Panel', 'Printers and Faxes', 'Help and Support', 'Search', 'Run...')
    $yy = 246; $k = 0
    foreach ($nm in $rn2) {
        $ci = $ic[($k + 4) % 6]
        RtBox $m 198 $yy 22 22 $ci 4
        RtText $m $nm 232 ($yy + 4) 12 (Brush 0 40 110) $false 'Tahoma'
        $yy += 32; $k++
    }
    # the bottom bar
    RtBox $m 118 433 26 26 (Brush 240 170 40) 5
    RtText $m 'Log Off' 150 438 12 $white $false 'Tahoma'
    RtBox $m 222 433 26 26 (Brush 214 64 40) 5
    RtText $m 'Turn Off Computer' 254 438 12 $white $false 'Tahoma'
    return $m
}
# the flash that covers the change (one gentle fade in and out, never a strobe)
function RetroEnsure {
    EnsureFx
    if ($null -ne $S.rt.flash) { return }
    $fr = New-Object Windows.Shapes.Rectangle
    $fr.Width = $vw; $fr.Height = $vh; $fr.Fill = (Brush 255 255 255); $fr.Opacity = 0.0; $fr.IsHitTestVisible = $false
    [Windows.Controls.Panel]::SetZIndex($fr, 50)
    [void]$S.lzc.Children.Add($fr)
    $S.rt.flash = $fr
}
# keeps the old look above your real taskbar (and him above the old look)
function RetroTop {
    try { $S.fxw.Topmost = $false; $S.fxw.Topmost = $true; $win.Topmost = $false; $win.Topmost = $true } catch { }
}
function RetroBegin($t) {
    if ($S.rt.state -ne 'off') { return }
    RetroEnsure
    $S.rt.state = 'in'; $S.rt.t0 = $t
}
# the flash is at its brightest: swap everything while nobody can see it
function RetroApply($t) {
    $r = $S.rt
    $r.end = $t + $RetroSeconds
    $r.topT = $t + 1.5; $r.clkT = $t + 1.0; $r.btnT = $t + 3.0
    try { RetroWall $true } catch { Write-Host ('StickBuddy: could not swap the wallpaper: ' + $_.Exception.Message) -ForegroundColor Yellow }
    $ti = RetroMakeTaskbar
    $r.tbi = $ti
    if ($null -ne $ti) {
        [void]$S.lzc.Children.Add($ti.cv)
        $r.tbTop = $ti.top
        $r.clk = $ti.clk
        RetroButtons
    } else {
        $r.clk = $null
        $r.tbTop = [Windows.SystemParameters]::PrimaryScreenHeight
    }
    $mn = RetroMakeMenu
    [Windows.Controls.Canvas]::SetLeft($mn, 0 - $vl); [Windows.Controls.Canvas]::SetTop($mn, $r.tbTop - 470 - $vt)
    [Windows.Controls.Panel]::SetZIndex($mn, 3)
    [void]$S.lzc.Children.Add($mn)
    $r.sm = $mn; $r.smUntil = $t + 4.5
    RetroTop
    $S.msg = 'welcome to 2001!'; $S.msgT = $t + 2.6
}
# take the old look away and put your wallpaper back
function RetroRevert {
    $r = $S.rt
    try { RetroWall $false } catch { }
    if ($null -ne $r.tbi) { $S.lzc.Children.Remove($r.tbi.cv); $r.tbi = $null }
    if ($null -ne $r.sm) { $S.lzc.Children.Remove($r.sm); $r.sm = $null }
    $r.btns.Clear(); $r.clk = $null
}
# used when he leaves (ESC or closing the window) so your wallpaper is always put back
function RetroRestoreNow {
    if ($S.rt.state -eq 'off' -and -not $S.rt.hasWall) { return }
    RetroRevert
    try { if ($null -ne $S.rt.flash) { $S.rt.flash.Opacity = 0.0 } } catch { }
    $S.rt.state = 'off'
}
function RetroFail($err) {
    if ($S.rtOk) { Write-Host ('StickBuddy: the 2001 zap hit a problem and was turned off: ' + $err.Exception.Message + ' (script line ' + $err.InvocationInfo.ScriptLineNumber + ')') -ForegroundColor Yellow }
    $S.rtOk = $false
    try { RetroRevert } catch { }
    try { if ($null -ne $S.rt.flash) { $S.rt.flash.Opacity = 0.0 } } catch { }
    $S.rt.state = 'off'
}
function RetroTick($t, $dt) {
    $r = $S.rt
    switch ($r.state) {
        'off' { }
        'in' {                                             # flash up, swap at the peak
            $k = Clamp01 (($t - $r.t0) / 0.3)
            $r.flash.Opacity = 0.85 * $k
            if ($k -ge 1) { RetroApply $t; $r.state = 'out'; $r.t0 = $t }
        }
        'out' {                                            # flash fades away: welcome to 2001
            $k = Clamp01 (($t - $r.t0) / 0.8)
            $r.flash.Opacity = 0.85 * (1 - $k)
            if ($k -ge 1) { $r.flash.Opacity = 0.0; $r.state = 'on' }
        }
        'on' {                                             # the old look is on: keep it fresh until the time is up
            if ($t -ge $r.end) {
                $r.state = 'rin'; $r.t0 = $t
                $S.msg = 'back to the future!'; $S.msgT = $t + 2.2
            } else {
                if ($null -ne $r.sm -and $t -gt $r.smUntil) {          # the Start menu closes by itself
                    $fk = Clamp01 (($t - $r.smUntil) / 0.35)
                    $r.sm.Opacity = 1 - $fk
                    if ($fk -ge 1) { $S.lzc.Children.Remove($r.sm); $r.sm = $null }
                }
                if ($t -ge $r.topT) { $r.topT = $t + 1.5; RetroTop }
                if ($t -ge $r.clkT) { $r.clkT = $t + 1.0; if ($null -ne $r.clk) { $r.clk.Text = (Get-Date).ToString('h:mm tt') } }
                if ($t -ge $r.btnT) { $r.btnT = $t + 3.0; RetroButtons }
            }
        }
        'rin' {                                            # flash up again, switch back at the peak
            $k = Clamp01 (($t - $r.t0) / 0.3)
            $r.flash.Opacity = 0.85 * $k
            if ($k -ge 1) { RetroRevert; $r.state = 'rout'; $r.t0 = $t }
        }
        'rout' {
            $k = Clamp01 (($t - $r.t0) / 0.8)
            $r.flash.Opacity = 0.85 * (1 - $k)
            if ($k -ge 1) { $r.flash.Opacity = 0.0; $r.state = 'off' }
        }
    }
}

# ---------- smooth poses (so he eases from one pose into the next instead of snapping) ----------
function SmoothPose($o, $dt) {
    $p = $S.pose
    foreach ($k in 'lean', 'crouch', 'lift', 'lift2', 'speed', 'ua', 'la', 'ul', 'll', 'free', 'legA', 'fdx', 'fdy', 'flk') {
        $rate = 16.0
        if ($k -eq 'lean') { $rate = 11.0 }
        elseif ($k -eq 'free') { $rate = 8.0 }
        elseif ($k -eq 'crouch') { $rate = 24.0 }
        elseif ($k -eq 'fdx' -or $k -eq 'fdy' -or $k -eq 'flk') { $rate = 9.0 }
        $p[$k] = $p[$k] + ($o[$k] - $p[$k]) * (1 - [Math]::Exp(-$dt * $rate))
    }
    $q = @{}
    foreach ($k in @($p.Keys)) { $q[$k] = $p[$k] }
    $q.build = $o.build; $q.run = $o.run
    return $q
}

# ---------- draw the figure for one frame ----------
function Draw($o) {
    $pg = $S.g
    $b = $o.build
    $p0 = Clamp01 ($b / 0.5)            # head pops in
    $p1 = Clamp01 (($b - 0.5) / 0.4)    # body draws downward
    $p2 = Clamp01 (($b - 0.9) / 0.5)    # legs grow
    $p3 = Clamp01 (($b - 1.4) / 0.5)    # arms grow
    $legAvg = ($pg.lR + $pg.lL) / 2     # missing legs lower his hips
    $hip = Pt $FX ($FY - 56 * [Math]::Cos(0.9 * $o.crouch) - $o.lift + 40 * (1 - $legAvg))
    $sn = [Math]::Sin($o.lean); $cs = [Math]::Cos($o.lean)
    $neck = Pt ($hip.X + 34 * $sn) ($hip.Y - 34 * $cs)
    $head = Pt ($neck.X + 20 * $sn) ($neck.Y - 20 * $cs)
    $sh = Mix $neck $hip 0.15
    $S.head = $head

    $geo = New-Object Windows.Media.StreamGeometry
    $c = $geo.Open()
    $hp = $p0 * $pg.head
    if ($hp -gt 0.02) {                 # big round head, no face
        $r = 22 * (1 - [Math]::Pow(1 - $hp, 3))
        $sz = [Windows.Size]::new($r, $r)
        $c.BeginFigure((Pt ($head.X - $r) $head.Y), $true, $true)
        $c.ArcTo((Pt ($head.X + $r) $head.Y), $sz, 0, $false, 'Clockwise', $true, $true)
        $c.ArcTo((Pt ($head.X - $r) $head.Y), $sz, 0, $false, 'Clockwise', $true, $true)
    }
    Chain $c @($neck, $hip) $p1
    $legR = $null; $legL = $null
    foreach ($sg in -1, 1) {
        $key = 'lR'; if ($sg -lt 0) { $key = 'lL' }
        $ph = $o.run + $(if ($sg -lt 0) { 0 } else { [Math]::PI })
        $foot = Pt ($FX + $sg * 8 + 18 * $o.speed * [Math]::Sin($ph)) ($FY - $o.lift2 - 14 * $o.speed * [Math]::Max(0.0, [Math]::Cos($ph)))
        $mid = Mix $hip $foot 0.5
        $knee = Pt ($mid.X + $sg * (3 + 14 * $o.crouch)) $mid.Y
        if ($o.free -gt 0.01) {         # in the air the legs hang (or trail behind him) instead of standing on the ground
            $la0 = $o.legA + $sg * (0.1 + 0.05 * [Math]::Sin($S.t * 6 + $sg))
            $ft2 = Pt ($hip.X + 56 * [Math]::Sin($la0)) ($hip.Y + 56 * [Math]::Cos($la0))
            $kn2 = Pt (($hip.X + $ft2.X) / 2 + 5 * $sg * [Math]::Cos($la0)) (($hip.Y + $ft2.Y) / 2 - 5 * $sg * [Math]::Sin($la0))
            $foot = Mix $foot $ft2 $o.free
            $knee = Mix $knee $kn2 $o.free
        }
        Chain $c (Curve $hip $knee $foot) ($p2 * $pg[$key])
        if ($sg -lt 0) { $legL = @($hip, $knee, $foot) } else { $legR = @($hip, $knee, $foot) }
    }
    $aR = Arm $sh 1 $o.ua $o.la
    $aL = Arm $sh (-1) $o.ul $o.ll
    Chain $c (Curve $sh $aR[0] $aR[1]) ($p3 * $pg.aR)
    Chain $c (Curve $sh $aL[0] $aL[1]) ($p3 * $pg.aL)
    $c.Close()
    $outline.Data = $geo; $inner.Data = $geo
    $S.jt = @{ head = $head; torso = @($neck, $hip); aR = @($sh, $aR[0], $aR[1]); aL = @($sh, $aL[0], $aL[1]); lR = $legR; lL = $legL }

    # fire jets on both hands (white-hot core, yellow, orange). They stream DOWN while he hovers and trail BEHIND him when he flies.
    $showFire = (-not $S.noFire) -and ($S.fire -gt 0.03) -and ($p3 -ge 1)
    if ($showFire) {
        try {
            $ca = [Math]::Cos($S.ang); $sa = [Math]::Sin($S.ang)
            $fdx = $o.fdx * $ca + $o.fdy * $sa                    # (the direction on the screen, turned into figure coordinates)
            $fdy = (0 - $o.fdx) * $sa + $o.fdy * $ca
            $fnn = [Math]::Sqrt($fdx * $fdx + $fdy * $fdy)
            if ($fnn -lt 0.01) { $fdx = 0.0; $fdy = 1.0 } else { $fdx = $fdx / $fnn; $fdy = $fdy / $fnn }
            $hands = @($aR[1], $aL[1]); $pres = @($pg.aR, $pg.aL); $i = 0
            foreach ($hh in $hands) {
                if ($pres[$i] -gt 0.95) {
                    $sd = $i * 1.9
                    $fk = 1 + 0.18 * [Math]::Sin($S.t * 19 + $sd) + 0.1 * [Math]::Sin($S.t * 31 + $sd * 2)
                    $len = (24 + 62 * $S.fire) * $fk * $o.flk
                    $fw = 6 + 8 * $S.fire
                    $sway = 3 * [Math]::Sin($S.t * 13 + $sd) + 2 * [Math]::Sin($S.t * 29 + $sd)
                    $x0 = $hh.X + $fdx * 3; $y0 = $hh.Y + $fdy * 3
                    $fl[$i].Data = (PlumeGeo $x0 $y0 $len $fw $sway $S.t $sd $fdx $fdy)
                    $fl[$i + 2].Data = (PlumeGeo $x0 $y0 ($len * 0.78) ($fw * 0.62) ($sway * 0.8) $S.t ($sd + 1) $fdx $fdy)
                    $fl[$i + 4].Data = (PlumeGeo $x0 $y0 ($len * 0.45) ($fw * 0.32) ($sway * 0.5) $S.t ($sd + 2) $fdx $fdy)
                    foreach ($q in 0, 2, 4) { $fl[$i + $q].Visibility = 'Visible' }
                } else {
                    foreach ($q in 0, 2, 4) { $fl[$i + $q].Visibility = 'Hidden' }
                }
                $i++
            }
        } catch {
            $S.noFire = $true
            foreach ($pp in $fl) { $pp.Visibility = 'Hidden' }
        }
    } else { foreach ($pp in $fl) { $pp.Visibility = 'Hidden' } }
}

# ---------- the show ----------
$timer = New-Object Windows.Threading.DispatcherTimer
$timer.Interval = [TimeSpan]::FromMilliseconds(16)
$timer.Add_Tick({
  try {
    $now = [DateTime]::UtcNow
    $dt = [Math]::Max(0.004, [Math]::Min(0.05, ($now - $S.last).TotalSeconds)); $S.last = $now
    $S.t += $dt; $t = $S.t
    if ((([NativeBits]::GetAsyncKeyState(0x1B)) -band 0x8000) -ne 0) { $win.Close(); return }
    $lb = (([NativeBits]::GetAsyncKeyState(0x01)) -band 0x8000) -ne 0
    $shift = (([NativeBits]::GetAsyncKeyState(0x10)) -band 0x8000) -ne 0

    $cur = CursorDip
    $S.cu = $cur
    $tx = $cur.X + 3; $ty = $cur.Y + 4                     # feet go on the pointer tip

    UpdateParts $t $dt
    if ($S.paintOk) { try { PaintUpdate $t $dt $cur $lb } catch { PaintFail $_ } }

    # Shift+click grabs him off your pointer (or off the Paint canvas, where the window never takes clicks)
    if (($S.phase -eq 'ride' -or $S.inPaint) -and $shift -and $lb -and -not $S.lb -and $S.phase -ne 'held' -and $S.phase -ne 'build') {
        if ($S.phase -eq 'ride' -or (NearBody $cur)) { DoGrab }
    }
    if ($S.phase -eq 'held') {
        if (-not $lb) { DoRelease }
        else {
            $S.x = $cur.X; $S.y = $cur.Y + 118            # he hangs from the pointer by his head
            [void]$S.hist.Add(@{ x = $cur.X; y = $cur.Y; t = $t })
            if ($S.hist.Count -gt 8) { $S.hist.RemoveAt(0) }
        }
    }
    $S.lb = $lb

    $o = @{ build = $t; lean = 0.0; crouch = 0.0; lift = 0.0; lift2 = 0.0; run = $S.run; speed = 0.0
            ua = 0.8; la = 0.2; ul = 0.8; ll = 0.2; free = 0.0; legA = 0.0; fdx = 0.0; fdy = 1.0; flk = 1.0 }
    $say = ''

    if (-not $S.canFly -and ($S.phase -in 'takeoff', 'flyto', 'flyland', 'hover')) {          # no hands, no fire: he drops out of the sky
        $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0; $S.spin = 0.0
        $S.msg = 'oops!'; $S.msgT = $t + 1.2
    }

    switch ($S.phase) {
        'build' { if ($t -ge 2.0) { $S.phase = 'wave' } }
        'wave' {
            $say = 'hi!'; $o.ua = 2.5; $o.la = 2.5 + 0.7 * [Math]::Sin($t * 10)
            if ($t -ge 6.0) { $S.phase = 'wander'; $S.wt = $t }
        }
        'wander' {                                         # his own life: strolling, waving, hopping, throwing paint
            $S.ang += ($S.to - $S.ang) * [Math]::Min(1.0, $dt * 12)
            $far = ($S.x -lt ($S.cl - 40)) -or ($S.x -gt ($S.cr + 40)) -or ($S.y -gt ($S.cf + 70))
            $gnd = GroundAt $S.x $S.y
            if ($S.angry -and $null -eq $S.trap) {         # he has been attacked: payback time
                if ($S.canFly) { $S.phase = 'takeoff'; $S.pt = $t } else { $S.phase = 'chase'; $S.pt = $t; $S.blkT = 0.0 }
            } elseif ($S.angry -and $Lasers -and $S.g.head -gt 0.5 -and $t -ge $S.lzFreeT) {      # caged and angry: he burns his way out
                StartBurn $t $cur
            } elseif ($far) {                              # not on his stage (Paint opened or closed): fly or drop back to it
                $S.tgx = [Math]::Max($S.cl, [Math]::Min($S.cr, $S.x)); $S.tgy = $S.cf - 70
                if ($S.canFly) { $S.phase = 'flyland'; $S.pt = $t }
                else { $S.x = $S.tgx; $S.y = $S.cf - 150; $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0 }
            } elseif ($S.y -lt ($gnd - 3)) {               # standing on thin air: fall
                $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0
            } else {
                $S.y += ($gnd - $S.y) * [Math]::Min(1.0, $dt * 30)
                if ($t -ge $S.wt) { PickWander }
                if ($S.pathOn -and $S.inPaint) { if ($S.wm -eq 'walk' -or $S.wm -eq 'hopwind') { $S.wm = 'idle' }; $S.wt = [Math]::Max($S.wt, $t + 0.4) }   # he stops to watch you draw
                $lf = 1.0
                if ($S.legs -eq 1) { $lf = 0.5 } elseif ($S.legs -eq 0) { $lf = 0.2 }
                # with Paint open and in front he pelts your pointer with paint
                if ($S.inPaint -and $S.pfg -and $S.canFly -and ($S.wm -ne 'throw') -and ($t -ge $S.nextItem)) {
                    $ar = $S.pr
                    if ($cur.X -ge $ar.L -and $cur.X -le $ar.R -and $cur.Y -ge $ar.T -and $cur.Y -le $ar.B -and (Dist $cur (Pt $S.x $S.y)) -gt 120) {
                        $S.wm = 'throw'; $S.tw0 = $t; $S.nb = 0
                        $S.big = ($rng.NextDouble() -lt 0.2)
                        if ($S.big) { $S.nbMax = 1 } else { $S.nbMax = 2 + $rng.Next(2) }
                        if ($S.angry) { $S.nbMax += 1 }
                        $S.wt = $t + 5
                    }
                }
                if ($S.wm -eq 'walk') {
                    $dx = $S.wx - $S.x
                    if ([Math]::Abs($dx) -lt 6) { $S.wt = $t }
                    else {
                        $dir = [Math]::Sign($dx); $spd = $WalkSpeed * $lf
                        $mvx = [Math]::Max($S.cl - $S.x, [Math]::Min($S.cr - $S.x, $dir * $spd * $dt))
                        $r = MoveBody $mvx 0.0
                        $S.face = $dir
                        if ($r -band 1) { $S.wm = 'idle'; $S.wt = $t + 0.5 + 1.5 * $rng.NextDouble() }      # a line you drew is in his way
                        else {
                            $S.run += $dt * $spd / 11
                            $o.run = $S.run; $o.lean = 0.1 * $dir
                            if ($S.legs -gt 0) { $o.speed = 0.9 }
                            $o.lift = 2.4 * [Math]::Abs([Math]::Sin($S.run))                                 # a little bounce in every step
                            $sw = 0.6 * [Math]::Sin($S.run)
                            $o.ua = 0.8 + $sw; $o.la = 0.2 + $sw; $o.ul = 0.8 - $sw; $o.ll = 0.2 - $sw
                        }
                    }
                } elseif ($S.wm -eq 'wave') {
                    $say = 'hi!'; $o.ua = 2.5; $o.la = 2.5 + 0.7 * [Math]::Sin($t * 10)
                } elseif ($S.wm -eq 'hopwind') {                                                             # wind up, then jump
                    $o.crouch = 0.9; $o.ua = 1.3; $o.la = 0.6; $o.ul = 1.3; $o.ll = 0.6
                    if ($t -ge $S.hopT) {
                        $S.wm = 'idle'; $S.wt = $t + 1.2
                        $S.phase = 'fly'; $S.soft = $true; $S.vx = 90.0 * (2 * $rng.Next(2) - 1); $S.vy = -560.0; $S.spin = 0.0
                    }
                } elseif ($S.wm -eq 'throw') {
                    $el = $t - $S.tw0
                    $o.ua = 2.9; $o.la = 2.4                                   # arm cocked back
                    if ($S.nb -gt 0 -and ($el - (0.45 + 0.45 * ($S.nb - 1))) -lt 0.2) { $o.ua = 0.9; $o.la = 0.4 }   # follow-through
                    if ($S.nb -lt $S.nbMax -and $el -ge (0.45 + 0.45 * $S.nb)) { SpawnBall $S.big; $S.nb += 1 }
                    if ($S.nb -ge $S.nbMax -and $el -ge (0.45 + 0.45 * ($S.nbMax - 1) + 0.3)) {
                        $S.wm = 'idle'; $S.wt = $t + 0.5
                        $S.nextItem = $t + 4 + 5 * $rng.NextDouble()
                        if ($S.angry -or $null -ne $S.trap) { $S.nextItem = $t + 2 + 2 * $rng.NextDouble() }
                    }
                } else {
                    $sw = 0.08 * [Math]::Sin($t * 1.7)                         # just standing around: a little sway and breathing
                    $o.ua = 0.8 + $sw; $o.la = 0.2 + $sw; $o.ul = 0.8 - $sw; $o.ll = 0.2 - $sw
                    $o.lift = 0.8 * [Math]::Sin($t * 2.2)
                }
            }
        }
        'chase' {                                          # no hands = no fire: he storms over on foot (or crawls)
            $S.ang += ($S.to - $S.ang) * [Math]::Min(1.0, $dt * 12)
            $lf = 1.0
            if ($S.legs -eq 1) { $lf = 0.5 } elseif ($S.legs -eq 0) { $lf = 0.2 }
            if ($null -ne $S.trap -or ($t - $S.pt) -gt 14) { $S.angry = $false; $S.phase = 'wander'; $S.wt = $t }
            else {
                $gnd = GroundAt $S.x $S.y
                if ($S.y -lt ($gnd - 3)) { $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0 }
                else {
                    $S.y += ($gnd - $S.y) * [Math]::Min(1.0, $dt * 30)
                    $dx = $cur.X - $S.x
                    if ([Math]::Abs($dx) -gt 110) {
                        $dir = [Math]::Sign($dx); $S.face = $dir
                        $mvx = [Math]::Max($S.cl - $S.x, [Math]::Min($S.cr - $S.x, $dir * 260 * $lf * $dt))
                        $r = MoveBody $mvx 0.0
                        if ($r -band 1) {                                              # a drawn wall stops him; after a while he gives up
                            if ($S.blkT -eq 0.0) { $S.blkT = $t } elseif (($t - $S.blkT) -gt 1.5) { $S.angry = $false; $S.phase = 'wander'; $S.wt = $t; $S.blkT = 0.0 }
                        } else { $S.blkT = 0.0; $S.run += $dt * 13 * $lf }
                        $o.run = $S.run; $o.lean = 0.12 * $dir
                        if ($S.legs -gt 0) { $o.speed = 0.9 }
                        $o.lift = 3 * [Math]::Abs([Math]::Sin($S.run))
                        $sw = 0.7 * [Math]::Sin($S.run)
                        $o.ua = 0.8 + $sw; $o.la = 0.2 + $sw; $o.ul = 0.8 - $sw; $o.ll = 0.2 - $sw
                    } elseif (($t - $S.pt) -gt 0.3) { $S.phase = 'crouch'; $S.pt = $t }
                }
            }
        }
        'crouch' {
            $say = 'grr!'; $k = Clamp01 (($t - $S.pt) / 0.5); $o.crouch = $k
            $o.ua = 1.3; $o.la = 0.6; $o.ul = 1.3; $o.ll = 0.6
            if ($k -ge 1) { $S.phase = 'hop'; $S.pt = $t; $S.sx = $S.x; $S.sy = $S.y; $S.hx = $cur.X; $S.hy = $cur.Y }
        }
        'hop' {
            $say = 'grr!'
            $u = Clamp01 (($t - $S.pt) / 1.2)
            $dist = [Math]::Sqrt([Math]::Pow($tx - $S.sx, 2) + [Math]::Pow($ty - $S.sy, 2))
            $nx = $S.sx + ($tx - $S.sx) * $u
            $ny = $S.sy + ($ty - $S.sy) * $u - (70 + 0.15 * $dist) * 4 * $u * (1 - $u)
            $r = 0
            if ((Dist $cur (Pt $S.hx $S.hy)) -le 30) { $r = MoveBody ($nx - $S.x) ($ny - $S.y) }
            if ((Dist $cur (Pt $S.hx $S.hy)) -gt 30 -or $r -ne 0) {                    # you moved the mouse (or a line is in the way): he drops down and tries again
                $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0; $S.spin = 0.0
            } else {
                $o.lift2 = 14 * [Math]::Sin([Math]::PI * $u); $o.crouch = 0.5 * [Math]::Sin([Math]::PI * $u)
                $a = 2.7 + 0.2 * [Math]::Sin($t * 14); $o.ua = $a; $o.la = $a; $o.ul = $a; $o.ll = $a
                if ($u -ge 1) { $S.phase = 'ride'; $S.angry = $false; $S.rideT = $t; $S.pt = $t; $S.still = $t; $S.lc = $cur }
            }
        }
        'ride' {                                           # stands on your pointer for a few seconds, then hops off
            $k = 1 - [Math]::Exp(-$dt * 14)
            $ox = $tx - $S.x; $oy = $ty - $S.y
            $S.x += $ox * $k; $S.y += $oy * $k
            $sp = Clamp01 (([Math]::Abs($ox) + [Math]::Abs($oy)) / 30)
            if ($sp -gt 0.08) { $S.run += $dt * (4 + 12 * $sp) }
            $o.speed = $sp; $o.run = $S.run
            $o.lean = [Math]::Max(-0.5, [Math]::Min(0.5, $ox / 80))
            $o.lift = 4 * $sp * [Math]::Abs([Math]::Sin($S.run * 2))
            $o.crouch = 0.8 * [Math]::Max(0.0, 1 - ($t - $S.pt) * 5)
            if (([Math]::Abs($cur.X - $S.lc.X) + [Math]::Abs($cur.Y - $S.lc.Y)) -gt 1.5) { $S.still = $t }
            $S.lc = $cur
            if (($t - $S.rideT) -lt 2.5) { $say = 'shift+drag me' }
            if (($t - $S.still) -gt 2.5) {                 # mouse sat still: wave again
                $o.ua = 2.5; $o.la = 2.5 + 0.7 * [Math]::Sin($t * 10)
            } else {
                $sw = 0.6 * $sp * [Math]::Sin($S.run)
                $o.ua = 0.8 + $sw; $o.la = 0.2 + $sw; $o.ul = 0.8 - $sw; $o.ll = 0.2 - $sw
            }
            if (($t - $S.rideT) -gt 6) {
                $S.phase = 'fly'; $S.soft = $true
                $S.vx = 200.0 * (2 * $rng.Next(2) - 1); $S.vy = -650.0; $S.spin = ($rng.NextDouble() - 0.5) * 6
            }
        }
        'takeoff' {                                        # power up (arms out, flames building), then launch
            $say = 'flame on!'; $k = Clamp01 (($t - $S.pt) / 0.45); $o.crouch = 0.8 * $k
            $o.ua = 1.0; $o.la = 0.1; $o.ul = 1.0; $o.ll = 0.1
            $o.fdx = 0.0; $o.fdy = 1.0; $o.flk = 1.2
            $fxd = $cur.X - $S.x; if ([Math]::Abs($fxd) -gt 10) { $S.face = [Math]::Sign($fxd) }
            if ($k -ge 1) { $S.phase = 'flyto'; $S.vx = 0.0; $S.vy = -420.0; $S.pt = $t }
        }
        'flyto' {                                          # fire jets from the hands: fly head first towards the pointer
            if ($Lasers -and $S.g.head -gt 0.5) {          # (he stops a little way off, to zap you)
                $gx = [Math]::Max($S.cl, [Math]::Min($S.cr, $tx - $S.face * 150))
                $gy = [Math]::Max($S.cc + 20, [Math]::Min($S.cf - 40, $ty - 10))
                $d = JetTo $gx $gy $dt $t $o
                if ($d -lt 40) { StartHover $t }
            } else {
                $d = JetTo $tx $ty $dt $t $o
                if ($d -lt 45) { $S.phase = 'ride'; $S.angry = $false; $S.rideT = $t; $S.pt = $t; $S.still = $t; $S.lc = $cur }
            }
            if (($t - $S.pt) -gt 7 -and $S.phase -eq 'flyto') {      # can't get there (walled in?): he gives up
                $S.angry = $false; $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0; $S.spin = 0.0
                $S.msg = 'meh.'; $S.msgT = $t + 1.2
            }
        }
        'hover' {                                          # eye-laser pose: hovering on his jets, beams on your pointer (and maybe a window)
            $S.ang += ($S.to - $S.ang) * [Math]::Min(1.0, $dt * 12)
            $k1 = [Math]::Min(1.0, $dt * 12)
            $ax2 = $tx; $ay2 = $ty
            if ($S.hz -eq 'retro') { $ax2 = 60.0; $ay2 = [Windows.SystemParameters]::PrimaryScreenHeight - 20 }      # the beams swing down to the Start button
            $S.aimX += ($ax2 - $S.aimX) * $k1; $S.aimY += ($ay2 - $S.aimY) * $k1
            $fxd = $S.aimX - $S.x; if ([Math]::Abs($fxd) -gt 25) { $S.face = [Math]::Sign($fxd) }
            $gx = [Math]::Max($S.cl, [Math]::Min($S.cr, $tx - $S.face * 150))
            $gy = [Math]::Max($S.cc + 20, [Math]::Min($S.cf - 40, $ty - 10))
            [void](JetTo $gx $gy $dt $t $o)
            LaserPose $o $true
            $el = $t - $S.hzt
            if ($S.hz -eq 'lock') {
                $say = 'pew pew!'
                if ($el -gt 1.25) {
                    if ($S.doRetro) { $S.hz = 'retro'; $S.hzt = $t; RetroBegin $t }
                    elseif ($S.doSlice -and (SliceStart $t)) { $S.hz = 'sweep'; $S.hzt = $t } else { EndHover $t $cur }
                }
            } elseif ($S.hz -eq 'retro') {
                $say = 'back to 2001!'
                if ($el -gt 1.9) { EndHover $t $cur }
            } elseif ($S.hz -eq 'sweep') {
                $say = 'snip!'
                $prog = Clamp01 ($el / 0.8)
                SliceSweep $prog
                if ($prog -ge 1) { SliceShatter $t; $S.hz = 'watch'; $S.hzt = $t; $S.lzOn = $false }
            } else {
                $say = 'ha!'
                if ($el -gt 1.1) { EndHover $t $cur }
            }
        }
        'burn' {                                           # caged: standing still, beams on the wall until it burns through
            $S.ang += ($S.to - $S.ang) * [Math]::Min(1.0, $dt * 12)
            if ($null -eq $S.trap -or $S.g.head -lt 0.3) { $S.lzOn = $false; $S.phase = 'wander'; $S.wt = $t }
            else {
                $gnd = GroundAt $S.x $S.y
                $S.y += ($gnd - $S.y) * [Math]::Min(1.0, $dt * 30)
                $k1 = [Math]::Min(1.0, $dt * 9)
                $S.aimX += ($S.bp.X - $S.aimX) * $k1; $S.aimY += ($S.bp.Y - $S.aimY) * $k1
                $fxd = $S.bp.X - $S.x; if ([Math]::Abs($fxd) -gt 15) { $S.face = [Math]::Sign($fxd) }
                LaserPose $o $false
                $say = 'LASERS!'
                if (($t - $S.pt) -gt 1.7) { BurnAt $S.bp.X $S.bp.Y; $S.lzOn = $false; $S.phase = 'wander'; $S.wt = $t }
            }
        }
        'flyland' {                                        # fly to a spot on his stage and drop in
            $d = JetTo $S.tgx $S.tgy $dt $t $o
            if ($d -lt 30 -or ($t - $S.pt) -gt 6) { $S.phase = 'fly'; $S.soft = $true; $S.vx = 0.0; $S.vy = 0.0; $S.spin = 0.0 }
        }
        'held' {                                           # dangling from your pointer
            $say = 'put me down!'; $S.run += $dt * 5; $o.speed = 0.5; $o.run = $S.run
            $o.ua = 1.7 + 0.4 * [Math]::Sin($S.run * 1.3); $o.la = $o.ua
            $o.ul = 1.7 + 0.4 * [Math]::Sin($S.run * 1.3 + 2); $o.ll = $o.ul
            $o.free = 1.0; $o.legA = 0.25 * [Math]::Sin($S.run * 1.1)
            $tg = 0.0
            if ($S.hist.Count -gt 1) { $tg = [Math]::Max(-0.6, [Math]::Min(0.6, ($S.hist[$S.hist.Count - 1].x - $S.hist[0].x) / 250)) }
            $S.ang += ($tg - $S.ang) * 0.15
        }
        'fly' {                                            # thrown (or hopping): gravity, bounces, tumbling, and the lines you drew are solid
            if (-not $S.soft) { $say = 'aaaah!' }
            $S.vy += 1900 * $dt; $S.ang += $S.spin * $dt; $S.run += $dt * 8
            $o.speed = 0.6; $o.run = $S.run
            $a = 2.6 + 0.5 * [Math]::Sin($S.run); $o.ua = $a; $o.la = $a; $o.ul = $a; $o.ll = $a
            if (-not $S.soft) { $o.free = 1.0; $o.legA = 0.3 * [Math]::Sin($S.run) }
            $r1 = MoveBody ($S.vx * $dt) 0.0
            if ($r1 -band 1) { $S.vx = (0 - $S.vx) * 0.5; $S.spin *= -0.5 }
            if ($S.x -lt $S.cl) { $S.x = $S.cl; $S.vx = [Math]::Abs($S.vx) * 0.6; $S.spin *= -0.6 }
            if ($S.x -gt $S.cr) { $S.x = $S.cr; $S.vx = (0 - [Math]::Abs($S.vx)) * 0.6; $S.spin *= -0.6 }
            $gl = GroundAt $S.x $S.y                       # the floor, or the top of a line he is falling onto
            $r2 = MoveBody 0.0 ($S.vy * $dt)
            if (($r2 -band 2) -and $S.vy -lt 0) { $S.vy = [Math]::Abs($S.vy) * 0.3 }
            if ($S.y -lt $S.cc) { $S.y = $S.cc; $S.vy = [Math]::Abs($S.vy) * 0.5 }
            if ($S.vy -ge 0 -and $S.y -ge $gl) {
                $S.y = $gl; $S.vy = (0 - [Math]::Abs($S.vy)) * 0.45; $S.vx *= 0.8; $S.spin *= 0.6
                if ([Math]::Abs($S.vy) -lt 220 -and [Math]::Abs($S.vx) -lt 80) {
                    $S.vx = 0.0; $S.vy = 0.0; $S.pt = $t; $S.gl = $gl; $S.sq = 1.0
                    if ($S.soft) { $S.phase = 'wander'; $S.wt = $t; $S.to = [Math]::Round($S.ang / (2 * [Math]::PI)) * 2 * [Math]::PI }
                    else { $S.phase = 'lie'; $S.to = [Math]::Round(($S.ang - [Math]::PI / 2) / [Math]::PI) * [Math]::PI + [Math]::PI / 2 }
                }
            }
        }
        'lie' {                                            # dazed on the floor for a second
            $say = 'ow.'; $k = [Math]::Min(1.0, $dt * 10)
            $S.ang += ($S.to - $S.ang) * $k; $S.y += (($S.gl + 44) - $S.y) * $k
            $o.ua = 0.6; $o.la = 0.6; $o.ul = 0.6; $o.ll = 0.6; $o.crouch = 0.2
            if (($t - $S.pt) -gt 1.0) { $S.phase = 'getup'; $S.pt = $t; $S.to = [Math]::Round($S.ang / (2 * [Math]::PI)) * 2 * [Math]::PI }
        }
        'getup' {
            $k = [Math]::Min(1.0, $dt * 9)
            $S.ang += ($S.to - $S.ang) * $k; $S.y += ($S.gl - $S.y) * $k
            if (($t - $S.pt) -gt 0.5) { $S.phase = 'wander'; $S.ang = 0.0; $S.to = 0.0; $S.y = $S.gl; $S.wt = $t }
        }
    }

    # landing squash: he crouches a little when he lands, then springs back
    $S.sq = $S.sq * [Math]::Exp(-$dt * 9)
    $o.crouch = [Math]::Min(1.1, $o.crouch + 0.55 * $S.sq)

    # fire when powering up, jumping, flying, held or thrown (no hands, no fire)
    $target = 0.0
    if ($S.canFly) {
        switch ($S.phase) { 'crouch' { $target = 0.6 } 'hop' { $target = 1.0 } 'takeoff' { $target = 0.9 } 'flyto' { $target = 1.0 } 'flyland' { $target = 1.0 } 'hover' { $target = 1.0 } 'held' { $target = 0.7 } 'fly' { if (-not $S.soft) { $target = 0.8 } } }
    }
    $S.fire += ($target - $S.fire) * [Math]::Min(1.0, $dt * 5)
    # the lasers only fire while he is hovering or burning; a slice that was interrupted just snaps apart
    if ($S.lzOn -and $S.phase -ne 'hover' -and $S.phase -ne 'burn') { $S.lzOn = $false }
    if ($null -ne $S.sl -and $S.sl.state -eq 'sweep' -and $S.phase -ne 'hover') { SliceShatter $t }
    Set-Through ($S.phase -eq 'ride' -or $S.inPaint)

    Draw (SmoothPose $o $dt)
    $rot.Angle = $S.ang * 180 / [Math]::PI
    $win.Left = $S.x - $FX; $win.Top = $S.y - $FY
    if ($t -lt $S.msgT -and $S.msg -ne '') { $say = $S.msg }
    if ($S.g.head -lt 0.3) { $say = '' }                   # no head, no talking
    if ($say) {
        $tb.Text = $say
        [Windows.Controls.Canvas]::SetLeft($bubble, $S.head.X + 30)
        [Windows.Controls.Canvas]::SetTop($bubble, $S.head.Y - 66)
        $bubble.Visibility = 'Visible'
    } else { $bubble.Visibility = 'Hidden' }

    # effects drawn on the second window: burn marks, laser beams, the sliced window
    try { HoleTick $t; LaserUpdate $t $dt; SliceUpdate $t $dt } catch {
        $S.lzOn = $false
        if (-not $S.lzErr) { $S.lzErr = $true; Write-Host ('StickBuddy: an effect failed: ' + $_.Exception.Message + ' (script line ' + $_.InvocationInfo.ScriptLineNumber + ')') -ForegroundColor Yellow }
        try { SliceEnd } catch { }
    }
    # the 2001 zap (kept apart on purpose: if it fails, only it is switched off, and your wallpaper is put back)
    if ($S.rtOk) { try { RetroTick $t $dt } catch { RetroFail $_ } }
    $S.pc = $cur
  } catch {
    if (-not $S.errShown) {
        $S.errShown = $true
        Write-Host ('StickBuddy hit a snag: ' + $_.Exception.Message + ' (script line ' + $_.InvocationInfo.ScriptLineNumber + ')') -ForegroundColor Yellow
    }
  }
})

$win.Add_SourceInitialized({
    $hw = (New-Object Windows.Interop.WindowInteropHelper $win).Handle
    $S.hwnd = $hw
    # tool window (0x80) + never takes focus (0x08000000); click-through (0x20) is switched on while he rides your pointer or Paint is open
    [void][NativeBits]::SetWindowLong($hw, -20, ([NativeBits]::GetWindowLong($hw, -20) -bor 0x08000000 -bor 0x80))
    $S.src = [Windows.PresentationSource]::FromVisual($win)
})
$win.Add_Loaded({ $S.last = [DateTime]::UtcNow; $timer.Start() })
$win.Add_Closed({
    $timer.Stop()
    try { RetroRestoreNow } catch { }                      # (always puts your own wallpaper back)
    try { if ($null -ne $S.fxw) { $S.fxw.Close() } } catch { }
})

Write-Host "StickBuddy is running. Leave him alone and he wanders; grab or throw him and he flies to your pointer, zaps you with lasers and slices your window - or zaps your desktop back to 2001 for 2 minutes. Open Paint for more. ESC (or close this window) makes him leave."
[void]$win.ShowDialog()
