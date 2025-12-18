---
title: VSCode编写Cpp使用system函数导致异常退出，解决办法
date: 2021-7-2
tags:
  - 年份-2021
  - 阶段-专科
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-VSCode
  - 主题-编程
---

## 问题描述

VSCode运行代码，默认是使用集成控制台的。

C++中调用的 `system()` 函数**可能**会导致程序异常退出。例如：

```cpp
/**
 * 主菜单界面显示
 */
short homeMenu()
{
	short code = 0;

	system("cls");

	cout << "------------------------"
		 << "通讯录管理系统"
		 << "------------------------" << endl
		 << endl;

	cout << "\t\t\t1、添加联系人" << endl
		 << "\t\t\t2、显示联系人" << endl
		 << "\t\t\t3、删除联系人" << endl
		 << "\t\t\t4、查找联系人" << endl
		 << "\t\t\t5、修改联系人" << endl
		 << "\t\t\t6、清空联系人" << endl
		 << "\t\t\t0、退出通讯录" << endl
		 << endl;

	cout << "请选择（0~6）：";
	cin >> code;
	getchar();
	cout << endl;

	return code;
}

```

提示信息：

![VSCode任务异常终止代码2](images/2021/VSCode任务异常终止代码2.png)

估计是集成控制台不支持这个函数。

## 解决办法

在 `.vscode/launch.json` 文件中，将

```json
"configurations": [
    {
        "externalConsole": false,
    }
]

```

改为

```json
"configurations": [
    {
        "externalConsole": true,
    }
]

```

使用生成黑窗口的系统控制台运行代码，即可解决。

CMD 乱码从根本上解决:

![Windows10系统全局修改UTF-8](images/2021/Windows10系统全局修改UTF-8.png)

