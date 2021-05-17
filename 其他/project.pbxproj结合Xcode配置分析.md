# project.pbxproj结合Xcode配置分析

## project.pbxproj层级

### PBXBuildFile
* 参与编译，链接、copy文件的PBXFileReference会有对应的PBXBuildFile，它会被PBXSourcesBuildPhase或PBXResourcesBuildPhase调用，这里一般不会有.h文件
* 对单个文件的 compiler flags ，存储在settings属性中，扩展的配置信息，对应 key/value 字典
* 示例：

```
DE7404451D3CA84C0077CD92 /* main.m in Sources */ = {
    isa = PBXBuildFile; 
    fileRef = DE7404441D3CA84C0077CD92 /* main.m */; 
    settings = {
       COMPILER_FLAGS = "-fno-objc-arc";
    };
};
```

### PBXFileReference
* PBXFileReference代表project所引用的所有文件，包括唯一标识码（UUID）、源文件、头文件、资源文件、库、Frameworks、xcconfig files、other projects、生成的应用文件等，它会被PBXGroup、PBXBuildFile等调用(除了Info.plist、.pch  通过路径确定)
* 包含两个类型的文件：input(源文件)---lastKnownFileType 和 output（.app .a .framework)--explicitFileType 
* lastKnownFileType：sourcecode.c.objc（.m）、sourcecode.c.h（.h）、sourcecode.swift(.swift)、file.xib、folder.assetcatalog（.xcassets）、file.storyboard（.storyboard）

* 示例：

```
DE7404441D3CA84C0077CD92 /* main.m */ = {
    isa = PBXFileReference; 
    lastKnownFileType = sourcecode.c.objc;
    path = main.m; /* 相对路径  （例如XCConfig工程：XCConfig/XCConfig/） 取决于sourceTree*/
    fileEncoding = 4; /*UTF-8*/
    sourceTree = "<group>"; /*相对的路径 可以使 ”“ 、SDKROOT、BUILT_PRODUCTS_DIR*/
};
```

### PBXFrameworksBuildPhase
* PBXFrameworksBuildPhase包含target依赖的系统及第三方的.framework和.a文件（build Phases-Link Binary With Libraries 或General-Linked Frameworks and Libraries）
* 如果是第三方library或Framework,则会XCBuildConfiguration中对应的target添加搜索路径（build Setting-Search Paths-Framework/Library Search Paths）

```
	LIBRARY_SEARCH_PATHS = (
					"$(inherited)",
					"$(PROJECT_DIR)/XCConfig",
				);
				
	FRAMEWORK_SEARCH_PATHS = (
					"$(inherited)",
					"$(PROJECT_DIR)/XCConfig",
				);
```
* 示例

```
DE74043D1D3CA84C0077CD92 /* Frameworks */ = {
		isa = PBXFrameworksBuildPhase;
		buildActionMask = 2147483647;
		files = (
		   DE8CE7101D3F94310051C320 /* libBase_lib.a in Frameworks */,
		   DE8CE70E1D3F94050051C320 /* JavaScriptCore.framework in Frameworks */,
			DE8CE6F21D3F84080051C320 /* Accounts.framework in Frameworks */,
		);
		runOnlyForDeploymentPostprocessing = 0;
	};
```


### PBXSourcesBuildPhase
* Project 通常有多个build phases:compiling,liking,copying resources,copying other files,run scripts。PBXSourcesBuildPhase描述了一个target的build phases
* 示例

```
	DE74043C1D3CA84C0077CD92 /* Sources */ = {
			isa = PBXSourcesBuildPhase;
			buildActionMask = 2147483647; /* == 0x7FFFFFFF */
			files = ( /*包含 PBXBuildFile中源文件的引用列表*/
				DE74044B1D3CA84C0077CD92 /* ViewController.m in Sources */,
				DE8CE70B1D3F90CA0051C320 /* TestDiskObject.m in Sources */,
				DE7404481D3CA84C0077CD92 /* AppDelegate.m in Sources */,
				DE7404451D3CA84C0077CD92 /* main.m in Sources */,
			);
			runOnlyForDeploymentPostprocessing = 0; /* FALSE */
		};
```

### XCConfigurationList
* XCConfigurationList 简单说就是一组configurations，一个configuration指的是一组设置,通常是：Debug和Release，前者用于调试项目，后者对用户体验上进行了优化。
* defaultConfigurationIsVisible和defaultConfigurationIsVisible决定这xcodebuild tool编译时的默认配置
* buildConfigurations 包含的子XCBuildConfiguration的引用

```	
DE74043B1D3CA84C0077CD92 /* Build configuration list for PBXProject "XCConfig" */ = {
			isa = XCConfigurationList;
			buildConfigurations = (
				DE7404551D3CA84C0077CD92 /* Debug */,
				DE7404561D3CA84C0077CD92 /* Release */,
			);
			defaultConfigurationIsVisible = 0;
			defaultConfigurationName = Release;
		};
```

### XCBuildConfiguration
 * XCBuildConfiguration 是Build Setting的集合，buildSettings是XCBuildConfiguration的核心，和.xconfig对应。
 * 继承关系  target 继承project

 * 示例
 
```
DE8CE7001D3F894A0051C320 /* Debug */ = {
			isa = XCBuildConfiguration;
			buildSettings = {
				ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon;
				ASSETCATALOG_COMPILER_LAUNCHIMAGE_NAME = "Brand Assets";
				CODE_SIGN_IDENTITY = "iPhone Developer";
				INFOPLIST_FILE = "XCConfig copy-Info.plist";
				IPHONEOS_DEPLOYMENT_TARGET = 7.0;
				LD_RUNPATH_SEARCH_PATHS = "$(inherited) @executable_path/Frameworks";
				PRODUCT_BUNDLE_IDENTIFIER = com.sina.XCConfig13;
				PRODUCT_NAME = "$(TARGET_NAME)";
				PROVISIONING_PROFILE = "a53912d2-fc78-4992-97ad-3be14b8eae10";
				TARGETED_DEVICE_FAMILY = "1,2";
			};
			name = Debug;
		};
```

### PBXVariantGroup
* PBXVariantGroup 描述了这样一组用于本地化的文件（strings、xib、storyboard），
* 示例

```
DE74044C1D3CA84C0077CD92 /* Main.storyboard */ = {
			isa = PBXVariantGroup;
			children = (
				DE74044D1D3CA84C0077CD92 /* Base */,
			);
			name = Main.storyboard;
			sourceTree = "<group>";
		};
		
116CDE3D17868D6300BCD47D /* InfoPlist.strings */ = {
			isa = PBXVariantGroup;
			children = (
				116CDE3E17868D6300BCD47D /* en */,
				11C238811786F39100DBADBD /* zh-Hans */,
			);
			name = InfoPlist.strings;
			sourceTree = "<group>";
		};
```





