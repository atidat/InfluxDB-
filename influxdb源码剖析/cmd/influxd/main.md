```go
var (
	version = "dev"
	commit  = "none"
	date    = ""
)

// 基于cobra实现的命令行，主要实现步骤包括
// 1、生成根命令rootCmd
// 2、添加子命令upgradeCmd、inspectCmd、vesionCmd、recoveryCmd和downgradeCmd
// 3、执行命令
func main() {
   if len(date) == 0 { date = time.Now().UTC().Format(time.RFC3339) }

   // 给influxd二进制文件，设置版本、commit内容和时间信息
   influxdb.SetBuildInfo(version, commit, date)

   ctx := context.Background()
   v := viper.New()

   // 1、生成根命令rootCmd
   rootCmd, err := launcher.NewInfluxdCommand(ctx, v)
   if err != nil { handleErr(err.Error()) }
    
   // 2、添加子命令upgradeCmd、inspectCmd、vesionCmd、recoveryCmd和downgradeCmd
   // upgrade binds options to env variables, so it must be added after rootCmd is initialized
   upgradeCmd, err := upgrade.NewCommand(ctx, v)
   if err != nil { handleErr(err.Error()) }
   rootCmd.AddCommand(upgradeCmd)
   inspectCmd, err := inspect.NewCommand(v)
   if err != nil { handleErr(err.Error()) }
   rootCmd.AddCommand(inspectCmd)
   rootCmd.AddCommand(versionCmd())
   rootCmd.AddCommand(recovery.NewCommand())
   downgradeCmd, err := downgrade.NewCommand(ctx, v)
   if err != nil { handleErr(err.Error()) }
   rootCmd.AddCommand(downgradeCmd)

   // 3、执行命令
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

