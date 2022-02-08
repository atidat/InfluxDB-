**cmd API**

NewInfluxdCommand(ctx context.Context, v *viper.Viper) (*cobra.Command, error)：生成influxd CLI，子命令"run"是默认的子命令

```
func NewInfluxdCommand(ctx context.Context, v *viper.Viper) (*cobra.Command, error) {
   o := NewOpts(v)
   cliOpts := o.BindCliOpts()

   prog := cli.Program{
      Name: "influxd",
      Run:  cmdRunE(ctx, o),
   }
   
   // 返回cobra.Command
   // 1、环境变量配置准备
   // 2、初始化配置（配置来源于flags、环境变量和配置文件之一）
   // 3、选项绑定（这一部分代码太复杂）
   cmd, err := cli.NewCommand(o.Viper, &prog)
   if err != nil { return nil, err }

   // Error out if invalid flags are found in the config file. This may indicate trying to launch 2.x using a 1.x config.
   if invalidFlags := invalidFlags(v); len(invalidFlags) > 0 {
      return nil, errInvalidFlags(invalidFlags, v.ConfigFileUsed())
   }

   runCmd := &cobra.Command{
      Use:  "run",
      RunE: cmd.RunE,
      Args: cobra.NoArgs,
   }
   for _, c := range []*cobra.Command{cmd, runCmd} {
      setCmdDescriptions(c)
      if err := cli.BindOptions(o.Viper, c, cliOpts); err != nil {
         return nil, err
      }
   }
   cmd.AddCommand(runCmd)
   printCmd, err := NewInfluxdPrintConfigCommand(v, cliOpts)
   if err != nil {
      return nil, err
   }
   cmd.AddCommand(printCmd)

   return cmd, nil
}
```