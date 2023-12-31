# new-email-from-file-watcher

Going by your description as well as your code I tried to reproduce the issue you're seeing without success. The only unknown seems to be "whatever is happening in the timer handler in `RadForm2`" but in this minimal example (where all it does is update the clock in the title bar) the timer is conspicuously running.

```
internal static class Program
{
    [STAThread]
    static void Main()
    {
        if (mutex.WaitOne(TimeSpan.Zero, true))
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);
            ApplicationConfiguration.Initialize();
            var path = Path.Combine(
                Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                Assembly.GetEntryAssembly().GetName().Name
            );
            Directory.CreateDirectory(path);

            // Open the folder so we can mess with the files.
            Process.Start(new ProcessStartInfo { FileName = path, UseShellExecute = true});

            using (var mainForm = new RadForm2())
            using(var fileWatcher = new FileSystemWatcher(path))
            {
                mainForm.MyTimer = new System.Windows.Forms.Timer
                {
                    Interval = 100,
                    Enabled = true,
                };
                mainForm.MyTimer.Tick += (sender, e) =>
                {
                    // In System Windows Forms.Timer, the Tick is already
                    // marshaled onto the UI thread and won't allow reentrancy.
                    // Other timers are less forgiving than that. To make sure
                    // that it's running, have it update a clock in the title bar.
                    if (mainForm.Visible) mainForm.Text = DateTime.Now.ToString();
                };
                fileWatcher.EnableRaisingEvents = true;
                fileWatcher.Created += (sender, e) => Debug.WriteLine($"{nameof(FileSystemWatcher.Created)}: {e.Name}");
                fileWatcher.Changed += (sender, e) => Debug.WriteLine($"{nameof(FileSystemWatcher.Changed)}: {e.Name}");
                fileWatcher.Renamed += (sender, e) => Debug.WriteLine($"{nameof(FileSystemWatcher.Renamed)}: {e.Name}");
                fileWatcher.Deleted += (sender, e) => Debug.WriteLine($"{nameof(FileSystemWatcher.Deleted)}: {e.Name}");
                mainForm.FormClosing += (sender, e) =>
                {
                    if (e.CloseReason == CloseReason.UserClosing)
                    {
                        e.Cancel = true;
                        mainForm.Hide();
                    }
                };
                NotifyIcon notifyIcon = new NotifyIcon();
                notifyIcon.Icon = Icon.ExtractAssociatedIcon(Application.ExecutablePath);
                notifyIcon.Text = "My App";
                notifyIcon.Visible = true;
                notifyIcon.DoubleClick += (sender, e) =>
                {
                    mainForm.Show();
                    mainForm.WindowState = FormWindowState.Normal;
                };
                ContextMenuStrip contextMenu = new ContextMenuStrip();
                contextMenu.Items.Add("Open", null, (sender, e) =>
                {
                    mainForm.Show();
                    mainForm.WindowState = FormWindowState.Normal;
                });
                contextMenu.Items.Add("Exit", null, (sender, e) =>
                {
                    notifyIcon.Visible = false;
                    mutex.ReleaseMutex();
                    Application.Exit();
                });
                notifyIcon.ContextMenuStrip = contextMenu;
                Application.Run();
            }
        }
        else
        {
            MessageBox.Show("Already running", "Alert", MessageBoxButtons.OK, MessageBoxIcon.Information);
        }
    }
    static Mutex mutex = new Mutex(true, "{3B50E02B-383F-4604-AF3E-6B79DD5B0926}");
}
```
___

There are three recommended changes in the code, but they shouldn't really have any bearing on the actual problem you're seeing.

- Moved the mutex release to the "Exit" handler so that it will do what you describe.
- Changed the `formClosing` so that it keeps the `hWnd` when the user is closing, but allows everything else (like disposing the window when the app closes).
- Wrapped the `IDisposable` instances in `using` blocks.