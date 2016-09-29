---
title: Android 主要的热修复方案原理分析
date: 2016-09-29 15:23:43
tags:
---
目前较为成熟的热修复框架主要有AndFix、Nuwa以及微信的热更新思想。现在将其主要思想总结如下：

# AndFix

[AndFix](https://github.com/alibaba/AndFix)是支付宝开源的一套热修复框架，使用简单，成功率高，基本满足大多数的bug修复场景。引入到项目中非常方便，主要分两步：

- 代码整合
  
  - build.gradle添加依赖 compile 'com.alipay.euler:andfix:0.4.0@aar'
  - Application.onCreate()方法中添加
    ```
    PatchManager patchManager = new PatchManager(this);
    patchManager.init(appversion);//current version
    patchManager.loadPatch();
    ```
    然后和后端协商一个补丁包下载服务器，在每次下载更新包到本地后
    ```
    patchManager.addPatch(path);
    ```
- 打补丁
  AndFix提供了一个打补丁包的工具，可以[去这里](https://raw.githubusercontent.com/alibaba/AndFix/master/tools/apkpatch-1.0.3.zip)下载，使用方法如下：
```
apkpatch -f <new> -t <old> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -f,--from <loc>        new Apk file path.
 -k,--keystore <loc>    keystore path.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
 -t,--to <loc>          old Apk file path.
```
AndFix的思想是直接更改修复的方法，具体我们可以看源码。先从PatchManager的init和load方法入手，这两个方法实现了补丁包的加载并最终调用了AndFixManager的fix方法：
```
for (Patch patch : mPatchs) {
			patchNames = patch.getPatchNames();
			if (patchNames.contains(patchName)) {
				classes = patch.getClasses(patchName);
				mAndFixManager.fix(patch.getFile(), classLoader, classes);
			}
		}
```
fix函数需要传入三个参数：patch文件、classloader以及需要fix的class列表。fix函数代码如下：
```
	/**
	 * fix
	 * 
	 * @param file
	 *            patch file
	 * @param classLoader
	 *            classloader of class that will be fixed
	 * @param classes
	 *            classes will be fixed
	 */
	public synchronized void fix(File file, ClassLoader classLoader,
			List<String> classes) {
		
		    .......省略........
            //load dex文件
			final DexFile dexFile = DexFile.loadDex(file.getAbsolutePath(),
					optfile.getAbsolutePath(), Context.MODE_PRIVATE);

			if (saveFingerprint) {
				mSecurityChecker.saveOptSig(optfile);
			}
           // 定义自己的classloader
			ClassLoader patchClassLoader = new ClassLoader(classLoader) {
				@Override
				protected Class<?> findClass(String className)
						throws ClassNotFoundException {
					Class<?> clazz = dexFile.loadClass(className, this);
					if (clazz == null
							&& className.startsWith("com.alipay.euler.andfix")) {
						return Class.forName(className);// annotation’s class
														// not found
					}
					if (clazz == null) {
						throw new ClassNotFoundException(className);
					}
					return clazz;
				}
			};
			Enumeration<String> entrys = dexFile.entries();
			Class<?> clazz = null;
			while (entrys.hasMoreElements()) {
				String entry = entrys.nextElement();
				if (classes != null && !classes.contains(entry)) {
					continue;// skip, not need fix
				}
				clazz = dexFile.loadClass(entry, patchClassLoader);
				if (clazz != null) {
					fixClass(clazz, classLoader);
				}
			}
		} catch (IOException e) {
			Log.e(TAG, "pacth", e);
		}
	}
```
这个方法的作用是load dex文件中的类，并依次修复，这个函数中有两处疑问：
- classes参数设计有何深意，因为我理解的patch包里面难道不都是需要修复的类吗，还会把没有修改的类打进去吗？
- 自定义了一个classloader,针对”com.alipay.euler.andfix"做了特殊处理，不知道怎么才会有这种场景。
针对这两个问题我特意咨询了AndFix的作者黎三平大神，大神给我的答复是：1. 这个设计有两个原因：
a） 新增类
b）早期patch工具打出的补丁包不是很准确
2.AndFix的一个注解，它的类加载会走到这来的。
大神的话还是不是很明白，大家如果看到了这块代码请帮我解释一下。

fix函数中遍历dex的类，并过滤掉不需要修复的类后调用fixclass函数，代码如下：
```
	private void fixClass(Class<?> clazz, ClassLoader classLoader) {
		Method[] methods = clazz.getDeclaredMethods();
		MethodReplace methodReplace;
		String clz;
		String meth;
		for (Method method : methods) {
			methodReplace = method.getAnnotation(MethodReplace.class);
			if (methodReplace == null)
				continue;
			clz = methodReplace.clazz();
			meth = methodReplace.method();
			if (!isEmpty(clz) && !isEmpty(meth)) {
				replaceMethod(classLoader, clz, meth, method);
			}
		}
	}
```
 代码很简洁，意思也很明了，就是找到这个类中需要修复的函数然后调用replaceMethod方法。替换方法在java层是无法做到的，所以这个函数最终还是调用了native的替换函数的方法，实质就是更改了类中方法所指向的地址，所以java不能做到。

jin的目录结构如下：
![AndFix Jni结构.png](http://upload-images.jianshu.io/upload_images/2023045-706b8167f9dc514a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

AndFix做了dalvik、art以及各平台的适配，核心是方法的替换，我们来看其中一个方法替换的函数：
```
extern void __attribute__ ((visibility ("hidden"))) dalvik_replaceMethod(
		JNIEnv* env, jobject src, jobject dest) {
	jobject clazz = env->CallObjectMethod(dest, jClassMethod);
	ClassObject* clz = (ClassObject*) dvmDecodeIndirectRef_fnPtr(
			dvmThreadSelf_fnPtr(), clazz);
	clz->status = CLASS_INITIALIZED;

	Method* meth = (Method*) env->FromReflectedMethod(src);
	Method* target = (Method*) env->FromReflectedMethod(dest);
	LOGD("dalvikMethod: %s", meth->name);

	meth->clazz = target->clazz;
	meth->accessFlags |= ACC_PUBLIC;
	meth->methodIndex = target->methodIndex;
	meth->jniArgInfo = target->jniArgInfo;
	meth->registersSize = target->registersSize;
	meth->outsSize = target->outsSize;
	meth->insSize = target->insSize;

	meth->prototype = target->prototype;
	meth->insns = target->insns;
	meth->nativeFunc = target->nativeFunc;
}
```
含义估计大家都明白，就是把方法的各个属性值替换，实际去写确实还是很有难度。

至此我把AndFix代码的主要流程梳理了一遍，其中还有很多没有get到的点，思想基本清楚了，AndFix主要采用替换方法的方式进行热修复，好处是立即生效且补丁包较小，但是只能基于方法修复，而且对平台的兼容性不佳，但不失为一个伟大的想法，也是热修复最早开源的修复方案，向我的黎三平大神说声感谢！

# Nuwa

AndFix的思路很简单，直接在native层替代方法，有没有更简单的呢，有，[Nuwa](https://github.com/jasonross/Nuwa)！他的想法就更自然一些，直接替换类，或者废弃掉有bug的类，怎么做到的呢，核心就在于java.lang.ClassLoader.java这个类的loadClass方法：
```
 protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }

            if (clazz == null) {
                try {
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }

        return clazz;
    }
```

同名的类加载一次就不再加载了，是不是想到什么了，哈哈，对，就是把你要修复的类提前加载就ok了，那么有bug的类便不再加载进来，实现起来也非常简单，三行代码搞定，我们看Nuwa的实现：
```
    public static void injectDexAtFirst(String dexPath, String defaultDexOptPath) throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
        //新建补丁包的dexclassloader
        DexClassLoader dexClassLoader = new DexClassLoader(dexPath, defaultDexOptPath, dexPath, getPathClassLoader());
        //获取原dex加载列表 
        Object baseDexElements = getDexElements(getPathList(getPathClassLoader()));
        // 新建补丁包dex加载列表
        Object newDexElements = getDexElements(getPathList(dexClassLoader));
        //补丁包插在最前连接成新的dex加载列表
        Object allDexElements = combineArray(newDexElements, baseDexElements);
        Object pathList = getPathList(getPathClassLoader());
        //利用发射修改dalvik.system.BaseDexClassLoader类的pathList中的dexElements属性
        ReflectionUtils.setField(pathList, pathList.getClass(), "dexElements", allDexElements);
    }
```
就这么几行代码就实现了替换类的热修复，没有native的操作，思路清晰，含义明确。当然这里面还有一个坑就是类加载的时候会有一个标记一个类和另外在同一个dex中的标记，所以打出补丁包之后会报 “两个类所在的dex不在一起” 的错误，这个也好办，打出一个单独的dex，里面只有一个类，让每个类都引用这个类，这样就使得每个类的标记都是false。Nuwa实现了这个插入引用单独类的插件，对使用者非常友好。

Nuwa能自由的修改和添加类，功能更加强大，不过这里面隐含了一个缺陷就是hook  classloader的dexElements必须在所有类没有加载之前进行，所以一般放在application的oncreate方法中，这样就导致了每次发布补丁必须重启app才能生效。

# 微信的热修复方案

其实技术方案的迭代也是思绪不断延伸的过程，看过Nuwa的热修复方案之后，是不是会想到有没有更简单更优雅的方式，有！其实很容易想到，我们在客户端实现补丁包的逻辑是什么呢，无非是比较两个dex中类文件发生更改的类提取出来，打成新的补丁dex，那这个过程能不能在客户端逆向操作一次呢，直接将差分包和原dex进行融合，形成新的dex，这样代码就不用做任何修改了，答案是肯定的。微信在一篇博客中阐述了自己热修复方案的主要思路([微信Android热补丁实践演进之路](http://www.tuicool.com/articles/uym2QrU))。没有具体实现，主要可能是文件权限的一些坑，但是像微信这样的app架构中，肯定是做了精准的分包处理，自己管理dex的加载策略，所以实现起来应该非常顺利。
微信在文中也坦言做热修复起源于15年6月，相对较晚，也可以综合比较之后设计出适合自己的方案。

以上是目前比较成熟的几个热修复方案，只是整理了主要思想，还有很多黑科技没有get到，希望对大家能有所帮助。