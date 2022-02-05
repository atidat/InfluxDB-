```go
var (
	version = "dev"
	commit  = "none"
	date    = ""
)

func main() {
   if len(date) == 0 { date = time.Now().UTC().Format(time.RFC3339) }

   // 给influx二进制文件，设置版本、commit内容和时间信息
   influxdb.SetBuildInfo(version, commit, date)

   ctx := context.Background()
   v := viper.New()

   rootCmd, err := launcher.NewInfluxdCommand(ctx, v)
   if err != nil {
      handleErr(err.Error())
   }
   // upgrade binds options to env variables, so it must be added after rootCmd is initialized
   upgradeCmd, err := upgrade.NewCommand(ctx, v)
   if err != nil {
      handleErr(err.Error())
   }
   rootCmd.AddCommand(upgradeCmd)
   inspectCmd, err := inspect.NewCommand(v)
   if err != nil {
      handleErr(err.Error())
   }
   rootCmd.AddCommand(inspectCmd)
   rootCmd.AddCommand(versionCmd())
   rootCmd.AddCommand(recovery.NewCommand())
   downgradeCmd, err := downgrade.NewCommand(ctx, v)
   if err != nil {
      handleErr(err.Error())
   }
   rootCmd.AddCommand(downgradeCmd)

   rootCmd.SilenceUsage = true
   if err := rootCmd.Execute(); err != nil {
      handleErr(fmt.Sprintf("See '%s -h' for help", rootCmd.CommandPath()))
   }
}

func handleErr(err string) {
   _, _ = fmt.Fprintln(os.Stderr, err)
   os.Exit(1)
}

func versionCmd() *cobra.Command {
   return &cobra.Command{
      Use:   "version",
      Short: "Print the influxd server version",
      Run: func(cmd *cobra.Command, args []string) {
         fmt.Printf("InfluxDB %s (git: %s) build_date: %s\n", version, commit, date)
      },
   }
}
```