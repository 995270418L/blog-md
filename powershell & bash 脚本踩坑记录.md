## powershell & bash 脚本踩坑记录

1. powershell 中执行外部程序（eg: exe程序）需要在语句前加 “&”。 eg:  

   >  & nginx.exe -s stop

2.  java 中执行powershell 获取正确的退出码执行命令

   > powershell -file xxxxx.ps1

3. java 中执行powershell路径必须是斜杠划分

   > powershell -file C:/steve/test.ps1

