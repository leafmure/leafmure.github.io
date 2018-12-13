# iOS Warning

### ld: warning: directory not found for option“XXXXXX
原因：从项目中删除了文件和文件夹的文件夹路径没有删除，编译器访问不到该路径而警告⚠️

解决方案：targets => Build Settings =>  Library Search Paths & Framework Search Paths，删掉编译报warning的路径。

### Block implicitly retains 'self'; explicitly mention 'self' to indicate this...警告
解决方案：Building Settings ->搜索implicit retain of 'self'
将对应的值改为NO。

### 消除CocoaPods时导入的第三方警告
inhibit_warnings参数的小问题
解决方案： 
1. 在podfile中加入inhibit_all_warnings! 。
2. pods -> targets -> 选择报警告的第三方 -> build settings 搜索inhibit_warnings，选择YES。
