# Gradle Source



> 关于gradle需要知道的内容如下
> 
> 1.gradle是一个java程序，所以它的本质就是一堆的jar包
> 
> 2.gradle的存放路径在gradle user home下，查看该参数的值如下。打开idea/Android studio -> File -> settings -> build tools -> gradle
> 
> 3.gradle的构建采用的是c/s架(client/server)。
> 
> 当我们使用gradlew 命令以后先会去执行gradle-wrapper.jar去配置gradle版本。
> 
> 然后用classLoader去加载.gradle路径下的引导程序打开client进程。
> 
> 然后client会触发开启server进程(也就是gradle中常见的daemon进程)，daemon进程默认会在后台一直等待client通过socket发送构建请求。
> 
> 当server收到了构建请求，他就会把构建这个任务放到线程池里面，然后开始调度执行，构建完成以后会通过socket通知client进程。
> 
> 最后client进程就挂了，server还在后台慢慢等待任务的到来。
> 
>  
> 
> 下文源码分析分为如下4个部分
> 
> gradle wrapper —— gradlew到调用gradle引导"程序"(其实是个jar包)
> 
> gradle 启动 —— gradle引导程序到gradle client进程到开启deamon进程
> 
> gradle 构建 —— deamon进程到构建开始
> 
> gradle 构建 生命周期 —— 构建开始到构建完成









# gradle wrapper

> 当我们new了一个gradle project的时候总是会生成以下的模板文件

<img src="https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220214212610.png" title="" alt="" data-align="center">

## 模板分析

- .gradle
  
  > 这个是gradle文件的缓存路径。

- build
  
  > 这个是gradle构建的默认输出路径

- gradle
  
  > 这个路径官方并没有给出对应的解释，在我这浅薄的记忆里，我也只知道这个路径是存放gradle wrapper的路径

- app
  
  > 这个是项目的默认模块的路径

- .gitignore
  
  > git的忽略文件。（防止你把自己的项目下的一些隐私给push进去了

- gradlew /gradlew.bat
  
  > 一个是shell脚本，一个是window上的批处理

- local.properties
  
  > 本地的一些配置信息

- settings.gradle
  
  > 整个项目的配置文件

## gradle wrapper

> 既然要分析gradle wrapper不会连gradle wrapper是什么都不知道吧？

gradle wrapper在这。

<img src="https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220214214051.png" title="" alt="" data-align="center">

gradle wrapper就中译就是gradle的包裹。

包裹了一层的gradle。有意思。

实际上呢他其实是一个版本管理器。

也就是负责管理整个项目的gradle版本的

> 为什么？为什么需要管理项目的版本？
> 
> 很简单。

想象一个场景如果我们不是使用的gradle wrapper，我们是直接使用的是Gradle。

试想一下，如果我们有很多个项目。

a项目需要gradle 6.0

b项目需要gradle 7.0

c项目需要gradle 7.4

那好问题来了，我们写了a项目后跑去写b项目会发现版本不兼容，这时候我们就得手动去下载项目对应的gradle版本。然后写了b以后去写c项目又不兼容，好又换。

问题出现了

- 如果我们没有下载gradle版本要手动去下载

- 就算我们下载好了我们还有手动的为对应的版本做切换

你可能会疑惑

> java等一众语言好像并不需要过多的配置，那是因为他们做了向后兼容的，语言这种东西没有兼容性那就会是恐怖的事情。项目上线了更新一下环境发现跑不起来了。很搞笑，

> gradle不同于编程语言，他的向后兼容没有想象中的那么好。因为只是一个构建工具，所谓工具就是越好用越好，而且迁移成本没有想象的那么高，所以呢一般情况下版本迭代和新特性出现地比较快。

所以扯了很多gradle wrapper就是为了解决项目gradle的版本兼容的问题。

它会依据我们的项目下的配置文件自动去找对应的gradle版本，如果没有下载还自动帮我们下载。

## 源码分析

> 没头绪是吧。
> 
> 好像也是。
> 
> 从哪里开始？
> 
> 我们使用gradle是从那里还是使用的？
> 
> ./gradlew build ?
> 
> ./gradlew tasks ?
> 
> 那就先看看gradlew文件

这不看没啥看了吓一跳

<img title="" src="https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220214215935.png" alt="" data-align="center">

```bash
"%JAVA_EXE%" %DEFAULT_JVM_OPTS% %JAVA_OPTS% %GRADLE_OPTS% "-Dorg.gradle.appname=%APP_BASE_NAME%" -classpath "%CLASSPATH%" org.gradle.wrapper.GradleWrapperMain %*
```

它运行了一个java jar包

看看classpath

```bash
set CLASSPATH=%APP_HOME%\gradle\wrapper\gradle-wrapper.jar
```

为了更见清晰的知道它究竟干了啥，我把这段执行的代码echo了一下

```bash
echo "%JAVA_EXE%" %DEFAULT_JVM_OPTS% %JAVA_OPTS% %GRADLE_OPTS% "-Dorg.gradle.appname=%APP_BASE_NAME%" -classpath "%CLASSPATH%" org.gradle.wrapper.GradleWrapperMain %*
```

 "D:\Compiler\JDK/bin/java.exe" "-Xmx64m" "-Xms64m"   "-Dorg.gradle.appname=gradlew" -classpath "E:\Code\Gradle\\gradle\wrapper\gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain build

输出是这样的

翻译过来就是

执行java程序

- 堆内存最大为64m，最小为64m（也就是固定为64m-Dorg.gradle.appname=gradlew这句话我也不知道有啥用）

- 程序运行的参数为build（我命令行输入的./gradlew build所以是build如果你打tasks就是tasks）

- 执行的jar包路径为gradle-wrapper.jar 

- main函数的位置为org.gradle.wrapper.GradleWrapperMain

挺好的。

### 利用idea调试

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220214221301.png)

好起来了

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220214221337.png)

锁定了整个main的位置

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220214221433.png)

然后加入JVM参数以及对应的函数参数

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215090222.png)

build.gradle中引入相关依赖

```kotlin
dependencies {
    implementation(fileTree(baseDir = "gradle/wrapper"){
        include("**/*.jar")
    })
}
```

编写测试函数

```kotlin
fun main(args: Array<String>) {
    GradleWrapperMain.main(args)
}
```

打断点

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215090649.png)

开始调试

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215091134.png)

### 流程分析

这是整体流程

```java
public static void main(String[] args) throws Exception {
        File wrapperJar = wrapperJar();
        File propertiesFile = wrapperProperties(wrapperJar);
        File rootDir = rootDir(wrapperJar);
        CommandLineParser parser = new CommandLineParser();
        parser.allowUnknownOptions();
        parser.option(new String[]{"g", "gradle-user-home"}).hasArgument();
        parser.option(new String[]{"q", "quiet"});
        SystemPropertiesCommandLineConverter converter = new SystemPropertiesCommandLineConverter();
        converter.configure(parser);
        ParsedCommandLine options = parser.parse(args);
        Properties systemProperties = System.getProperties();
        systemProperties.putAll(converter.convert(options, new HashMap()));
        File gradleUserHome = gradleUserHome(options);
        addSystemProperties(systemProperties, gradleUserHome, rootDir);
        Logger logger = logger(options);
        WrapperExecutor wrapperExecutor = WrapperExecutor.forWrapperPropertiesFile(propertiesFile);
        wrapperExecutor.execute(args, new Install(logger, new Download(logger, "gradlew", "0"), new PathAssembler(gradleUserHome, rootDir)), new BootstrapMainStarter());
    }
```

第一步先把当前的jar路径包装成一个File文件

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215091731.png)

紧接着就是包装gradle.properties文件

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215091841.png)

再者就是new了一个CommandLineParser不知道是怎么个解析法。反正是用来解析的。

后又进行了一段配置。

```java
parser.allowUnknownOptions();
```

或许是怕我们乱传入参数报错吧。

然后去解析了一堆参数

```java
parser.option(new String[]{"g", "gradle-user-home"}).hasArgument();
        parser.option(new String[]{"q", "quiet"})
SystemPropertiesCommandLineConverter converter = new SystemPropertiesCommandLineConverter();
        converter.configure(parser);
        ParsedCommandLine options = parser.parse(args);
        Properties systemProperties = System.getProperties();
        systemProperties.putAll(converter.convert(options, new HashMap()));;
```

然后去拿了gradleUserHome的位置

```java
File gradleUserHome = gradleUserHome(options);
```

如果我们我们在cmd中给出了gradle user home那么直接就使用该路径。

如果没有那就调用GradleUserHomeLookup.gradleUserHome();获取

```java
private static File gradleUserHome(ParsedCommandLine options) {
        return options.hasOption("g") ? new File(options.option("g").getValue()) : GradleUserHomeLookup.gradleUserHome();
    }
```

先判断properties文件中有没有什么路径。

如果没有就去拿系统环境变量，如果还没拿到就使用默认路径

```java
public static File gradleUserHome() {
        String gradleUserHome;
        if ((gradleUserHome = System.getProperty("gradle.user.home")) != null) {
            return new File(gradleUserHome);
        } else {
            return (gradleUserHome = System.getenv("GRADLE_USER_HOME")) != null ? new File(gradleUserHome) : new File(DEFAULT_GRADLE_USER_HOME);
        }
    }
```

```java
 public static final String DEFAULT_GRADLE_USER_HOME = System.getProperty("user.home") + "/.gradle";
```

gradle user home的参数的优先级如下

cmd参数 > properties参数 > 系统环境变量

所以对于C盘爆红的小伙伴如果想自定义gradle user home可以尝试加一个系统环境变量

GRADLE_USER_HOME。

这是最方便最持久的一种解决方案了。

所以上述一大段流程都是在解析这解析那，其实没多大价值

现在来点不一样的

```java
Logger logger = logger(options);
        WrapperExecutor wrapperExecutor = WrapperExecutor.forWrapperPropertiesFile(propertiesFile);
        wrapperExecutor.execute(args, new Install(logger, new Download(logger, "gradlew", "0"), new PathAssembler(gradleUserHome, rootDir)), new BootstrapMainStarter());
```

logger就不说了。

wrapperExecutor确保wrapper下的properties文件安在

```java
 public static WrapperExecutor forWrapperPropertiesFile(File propertiesFile) {
        if (!propertiesFile.exists()) {
            throw new RuntimeException(String.format("Wrapper properties file '%s' does not exist.", propertiesFile));
        } else {
            return new WrapperExecutor(propertiesFile, new Properties());
        }
    }
```

确保之后就开始执行了

```java
 public void execute(String[] args, Install install, BootstrapMainStarter bootstrapMainStarter) throws Exception {
        File gradleHome = install.createDist(this.config);
        bootstrapMainStarter.start(args, gradleHome);
    }
```

这execute放了很多的参数

第一行createDist做了一件事情那就是帮我们配置gradle

- 如果我们的gradle user home下有properties文件对应的gradle版本那他就直接返回当前版本的gradle路径

- 如果没有那就依据properties下的下载路径下载一份然后解压。

```java
public File createDist(final WrapperConfiguration configuration) throws Exception {
        final URI distributionUrl = configuration.getDistribution();
        final String distributionSha256Sum = configuration.getDistributionSha256Sum();
        LocalDistribution localDistribution = this.pathAssembler.getDistribution(configuration);
        final File distDir = localDistribution.getDistributionDir();
        final File localZipFile = localDistribution.getZipFile();
        return (File)this.exclusiveFileAccessManager.access(localZipFile, new Callable<File>() {
            public File call() throws Exception {
                File markerFile = new File(localZipFile.getParentFile(), localZipFile.getName() + ".ok");
                if (distDir.isDirectory() && markerFile.isFile()) {
                    Install.InstallCheck installCheck = Install.this.verifyDistributionRoot(distDir, distDir.getAbsolutePath());
                    if (installCheck.isVerified()) {
                        return installCheck.gradleHome;
                    }

                    System.err.println(installCheck.failureMessage);
                    markerFile.delete();
                }

                boolean needsDownload = !localZipFile.isFile();
                URI safeDistributionUrl = Download.safeUri(distributionUrl);
                if (needsDownload) {
                    File tmpZipFile = new File(localZipFile.getParentFile(), localZipFile.getName() + ".part");
                    tmpZipFile.delete();
                    Install.this.logger.log("Downloading " + safeDistributionUrl);
                    Install.this.download.download(distributionUrl, tmpZipFile);
                    tmpZipFile.renameTo(localZipFile);
                }

                List<File> topLevelDirs = Install.this.listDirs(distDir);
                Iterator var5 = topLevelDirs.iterator();

                while(var5.hasNext()) {
                    File dir = (File)var5.next();
                    Install.this.logger.log("Deleting directory " + dir.getAbsolutePath());
                    Install.this.deleteDir(dir);
                }

                Install.this.verifyDownloadChecksum(configuration.getDistribution().toString(), localZipFile, distributionSha256Sum);

                try {
                    Install.this.unzip(localZipFile, distDir);
                } catch (IOException var7) {
                    Install.this.logger.log("Could not unzip " + localZipFile.getAbsolutePath() + " to " + distDir.getAbsolutePath() + ".");
                    Install.this.logger.log("Reason: " + var7.getMessage());
                    throw var7;
                }

                Install.InstallCheck installCheckx = Install.this.verifyDistributionRoot(distDir, safeDistributionUrl.toString());
                if (installCheckx.isVerified()) {
                    Install.this.setExecutablePermissions(installCheckx.gradleHome);
                    markerFile.createNewFile();
                    return installCheckx.gradleHome;
                } else {
                    throw new RuntimeException(installCheckx.failureMessage);
                }
            }
        });
    }
```

之后呢就开启了gradle进行构建

```java
bootstrapMainStarter.start(args, gradleHome);
```

```java
public void start(String[] args, File gradleHome) throws Exception {
        File gradleJar = findLauncherJar(gradleHome);
        if (gradleJar == null) {
            throw new RuntimeException(String.format("Could not locate the Gradle launcher JAR in Gradle distribution '%s'.", gradleHome));
        } else {
            URLClassLoader contextClassLoader = new URLClassLoader(new URL[]{gradleJar.toURI().toURL()}, ClassLoader.getSystemClassLoader().getParent());
            Thread.currentThread().setContextClassLoader(contextClassLoader);
            Class<?> mainClass = contextClassLoader.loadClass("org.gradle.launcher.GradleMain");
            Method mainMethod = mainClass.getMethod("main", String[].class);
            mainMethod.invoke((Object)null, args);
            if (contextClassLoader instanceof Closeable) {
                contextClassLoader.close();
            }

        }
    }
```

在gradle路径下搜索gradle-launcher-.*\\.jar

```java
static File findLauncherJar(File gradleHome) {
        File libDirectory = new File(gradleHome, "lib");
        if (libDirectory.exists() && libDirectory.isDirectory()) {
            File[] launcherJars = libDirectory.listFiles(new FilenameFilter() {
                public boolean accept(File dir, String name) {
                    return name.matches("gradle-launcher-.*\\.jar");
                }
            });
            if (launcherJars != null && launcherJars.length == 1) {
                return launcherJars[0];
            }
        }

        return null;
    }
```

应该是这玩意了

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215102634.png)

然后它把这个jar包加载进来

```java
URLClassLoader contextClassLoader = new URLClassLoader(new URL[]{gradleJar.toURI().toURL()}, ClassLoader.getSystemClassLoader().getParent());
            Thread.currentThread().setContextClassLoader(contextClassLoader);
            Class<?> mainClass = contextClassLoader.loadClass("org.gradle.launcher.GradleMain");
            Method mainMethod = mainClass.getMethod("main", String[].class);
            mainMethod.invoke((Object)null, args);
            if (contextClassLoader instanceof Closeable) {
                contextClassLoader.close();
            }
```

然后又调用了org.gradle.launcher.GradleMain的main方法。

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220215102854.png)

### 小结

对于wrapper而言它没做多少事情也就是根据我们的一些配置参数帮我们安装配置对应的gradle版本。做到了gradle的版本管理。除此之外他还帮我们启动了gradle。

# gradle启动

> Time: 2022-2-21

## 源码分析环境

> 首先把gradle源码给pull下来，方便后续的分析

## gradle启动流程

先找到调用处。

<img title="" src="https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220221114816.png" alt="" data-align="center">

先判断了java的版本确保在jdk1.8以上

然后反射类`org.gradle.launcher.bootstrap.ProcessBootstrap`的run方法。

调用了runNoExit

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220221120153.png)

这里加载了一堆的classpath和classloader

然后反射调用了传入的class

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220221120529.png)

调用了Main的run方法

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220221121007.png)

run方法

```java
public void run(String[] args) {
        RecordingExecutionListener listener = new RecordingExecutionListener();
        try {
            doAction(args, listener);
        } catch (Throwable e) {
            createErrorHandler().execute(e);
            listener.onFailure(e);
        }

        Throwable failure = listener.getFailure();
        ExecutionCompleter completer = createCompleter();
        if (failure == null) {
            completer.complete();
        } else {
            completer.completeWithFailure(failure);
        }
    }
```

run首先调用了doAction

```java
 @Override
    protected void doAction(String[] args, ExecutionListener listener) {
        createActionFactory().convert(Arrays.asList(args)).execute(listener);
    }
```

然后调用convert new了一个`WithLogging`

```java
@Override
    public CommandLineExecution convert(List<String> args) {
        ServiceRegistry loggingServices = createLoggingServices();

        LoggingConfiguration loggingConfiguration = new DefaultLoggingConfiguration();

        return new WithLogging(loggingServices,
            args,
            loggingConfiguration,
            new ParseAndBuildAction(loggingServices, args),
            new BuildExceptionReporter(loggingServices.get(StyledTextOutputFactory.class), loggingConfiguration, clientMetaData()));
    }
```

注意紧跟`ParseAndBuildAction`(因为它内部间接封装了一个构建任务的runnable。这也是debug了半天才知道的结论)

然后调用了execute（这里为了简洁舍去了不那么重要的配置

```java
@Override
        public void execute(ExecutionListener executionListener) {

            ......
            ......（省略了commandLine的参数配置，以及一些log的配置

            try {
                Action<ExecutionListener> exceptionReportingAction =
                    new ExceptionReportingAction(reporter, loggingManager,
                        new NativeServicesInitializingAction(buildLayout, loggingConfiguration, loggingManager,
                            new WelcomeMessageAction(buildLayout,
                                new DebugLoggerWarningAction(loggingConfiguration, action))));
                exceptionReportingAction.execute(executionListener);
            } finally {
                loggingManager.stop();
            }
        }
```

然后分析这个嵌套了4层的Action(看着都头大)。

讲个思路，代码就不贴了

> 这个4个action是一个组合式的结构，类似链表，一层里面包了一层。
> 
> 当调用exceptionReportingAction的execute首先会执行当前Action需要执行的逻辑，然后再执行内部组合的action的逻辑，这样一层层调用到最内层。

前面说要紧跟ParseAndBuildAction。而这个exceptionReportingAction的最内层就是

它。

又要换分析场景了

以下是`ParseAndBuildAction`的execute方法。

```java
@Override
        public void execute(ExecutionListener executionListener) {
            List<CommandLineAction> actions = new ArrayList<CommandLineAction>();
            actions.add(new BuiltInActions());
            createActionFactories(loggingServices, actions);

            CommandLineParser parser = new CommandLineParser();
            for (CommandLineAction action : actions) {
                action.configureCommandLineParser(parser);
            }

            Action<? super ExecutionListener> action;
            try {
                ParsedCommandLine commandLine = parser.parse(args);
                action = createAction(actions, parser, commandLine);
            } catch (CommandLineArgumentException e) {
                action = new CommandLineParseFailureAction(parser, e);
            }

            action.execute(executionListener);
        }
```

首先加了一个`BuiltInActions`到action数组里面。（**注意紧跟BuiltInActions**）

然后通过调用createActionsFactories再加入点action（注意BuildActionsFactory）

```java
@VisibleForTesting
    protected void createActionFactories(ServiceRegistry loggingServices, Collection<CommandLineAction> actions) {
        actions.add(new BuildActionsFactory(loggingServices));
    }
```

然后调用了配置，这里我们不管

然后它将这个actions数组转化为一个Action，最后调用执行

```java
Action<? super ExecutionListener> action;
            try {
                ParsedCommandLine commandLine = parser.parse(args);
                action = createAction(actions, parser, commandLine);
            } catch (CommandLineArgumentException e) {
                action = new CommandLineParseFailureAction(parser, e);
            }

            action.execute(executionListener);
```

先看看如何将一个actions这个arrayList转化为Action的

```java
private Action<? super ExecutionListener> createAction(Iterable<CommandLineAction> factories, CommandLineParser parser, ParsedCommandLine commandLine) {
            for (CommandLineAction factory : factories) {
                Runnable action = factory.createAction(parser, commandLine);
                if (action != null) {
                    return Actions.toAction(action);
                }
            }
            throw new UnsupportedOperationException("No action factory for specified command-line arguments.");
        }
```

先遍历这个list，然后调用每个元素的`createAction`

还记得加入的元素是什么吧，

- BuiltInActions

- BuildActionsFactory

BuiltInActions范围不为空的时候是commandLine的参数是help以及version的时候。

所以这里不是build的逻辑开启处。

这下把./gradlew help和./gradlew version的执行处找到了。不过这不是重点，先pass

```java
@Override
        public Runnable createAction(CommandLineParser parser, ParsedCommandLine commandLine) {
            if (commandLine.hasOption(HELP)) {
                return new ShowUsageAction(parser);
            }
            if (commandLine.hasOption(VERSION)) {
                return new ShowVersionAction();
            }
            return null;
        }
```

开始分析BuildActionsFactory

```java
 @Override
    public Runnable createAction(CommandLineParser parser, ParsedCommandLine commandLine) {
        Parameters parameters = parametersConverter.convert(commandLine, null);

        parameters.getDaemonParameters().applyDefaultsFor(jvmVersionDetector.getJavaVersion(parameters.getDaemonParameters().getEffectiveJvm()));

        if (parameters.getDaemonParameters().isStop()) {
            return stopAllDaemons(parameters.getDaemonParameters());
        }
        if (parameters.getDaemonParameters().isStatus()) {
            return showDaemonStatus(parameters.getDaemonParameters());
        }
        if (parameters.getDaemonParameters().isForeground()) {
            DaemonParameters daemonParameters = parameters.getDaemonParameters();
            ForegroundDaemonConfiguration conf = new ForegroundDaemonConfiguration(
                UUID.randomUUID().toString(), daemonParameters.getBaseDir(), daemonParameters.getIdleTimeout(), daemonParameters.getPeriodicCheckInterval(), fileCollectionFactory);
            return new ForegroundDaemonAction(loggingServices, conf);
        }
        if (parameters.getDaemonParameters().isEnabled()) {
            return runBuildWithDaemon(parameters.getStartParameter(), parameters.getDaemonParameters());
        }
        if (canUseCurrentProcess(parameters.getDaemonParameters())) {
            return runBuildInProcess(parameters.getStartParameter(), parameters.getDaemonParameters());
        }

        return runBuildInSingleUseDaemon(parameters.getStartParameter(), parameters.getDaemonParameters());
    }
```

由于gradle为了加大使用效率通常情况下build都会开启一个守护线程。

只要以前build过都能看到有守护线程在等待

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220221142322.png)

（默认情况下是开启的守护进程所）以会进入上述的一个if

```java
if (parameters.getDaemonParameters().isEnabled()) {
            return runBuildWithDaemon(parameters.getStartParameter(), parameters.getDaemonParameters());
        }
```

调用runBuildWithDaemo

这注释已近暴露太多老了

```java
private Runnable runBuildWithDaemon(StartParameterInternal startParameter, DaemonParameters daemonParameters) {
        // Create a client that will match based on the daemon startup parameters.
        ServiceRegistry clientSharedServices = createGlobalClientServices(true);
        ServiceRegistry clientServices = clientSharedServices.get(DaemonClientFactory.class).createBuildClientServices(loggingServices.get(OutputEventListener.class), daemonParameters, System.in);
        DaemonClient client = clientServices.get(DaemonClient.class);
        return runBuildAndCloseServices(startParameter, daemonParameters, client, clientSharedServices, clientServices);
    }
```

创建了一个全局的守护进程,不去看它怎么new的知道new出来就行了。

然后调用了runBuildAndCloseServices,返回了一个Runnable

```java
 private Runnable runBuildAndCloseServices(StartParameterInternal startParameter, DaemonParameters daemonParameters, BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer, ServiceRegistry sharedServices, Object... stopBeforeSharedServices) {
        BuildActionParameters parameters = createBuildActionParameters(startParameter, daemonParameters);
        Stoppable stoppable = new CompositeStoppable().add(stopBeforeSharedServices).add(sharedServices);
        return new RunBuildAction(executer, startParameter, clientMetaData(), getBuildStartTime(), parameters, sharedServices, stoppable);
    }
```

`RunBuildAction如下`

```java
public class RunBuildAction implements Runnable {
    private final BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer;
    private final StartParameterInternal startParameter;
    private final BuildClientMetaData clientMetaData;
    private final long startTime;
    private final BuildActionParameters buildActionParameters;
    private final ServiceRegistry sharedServices;
    private final Stoppable stoppable;

    public RunBuildAction(BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer, StartParameterInternal startParameter, BuildClientMetaData clientMetaData, long startTime,
                          BuildActionParameters buildActionParameters, ServiceRegistry sharedServices, Stoppable stoppable) {
        this.executer = executer;
        this.startParameter = startParameter;
        this.clientMetaData = clientMetaData;
        this.startTime = startTime;
        this.buildActionParameters = buildActionParameters;
        this.sharedServices = sharedServices;
        this.stoppable = stoppable;
    }

    @Override
    public void run() {
        try {
            BuildActionResult result = executer.execute(
                new ExecuteBuildAction(startParameter),
                buildActionParameters,
                new DefaultBuildRequestContext(new DefaultBuildRequestMetaData(clientMetaData, startTime, sharedServices.get(ConsoleDetector.class).isConsoleInput()), new DefaultBuildCancellationToken(), new NoOpBuildEventConsumer())
            );
            if (result.hasFailure()) {
                // Don't need to unpack the serialized failure. It will already have been reported and is not used by anything downstream of this action.
                throw new ReportedException();
            }
        } finally {
            if (stoppable != null) {
                stoppable.stop();
            }
        }
    }
}
```

runnable返回了就是创建Action了

```java
for (CommandLineAction factory : factories) {
                Runnable action = factory.createAction(parser, commandLine);
                if (action != null) {
                    return Actions.toAction(action);
                }
            }
```

```java
public static <T> Action<T> toAction(@Nullable Runnable runnable) {
        //TODO SF this method accepts Closure instance as parameter but does not work correctly for it
        if (runnable == null) {
            return Actions.doNothing();
        } else {
            return new RunnableActionAdapter<T>(runnable);
        }
    }
```

创建一个RunnableActionAdapter返回

createAction的职责完成了

紧接着就是run

```java
action.execute(executionListener);
```

execute直接调用runnable的run

```java
private static class RunnableActionAdapter<T> implements Action<T> {
        private final Runnable runnable;

        private RunnableActionAdapter(Runnable runnable) {
            this.runnable = runnable;
        }

        @Override
        public void execute(T t) {
            runnable.run();
        }

        @Override
        public String toString() {
            return "RunnableActionAdapter{runnable=" + runnable + "}";
        }
    }
```

也就是`RunBuildAction`的run方法

```java
@Override
    public void run() {
        try {
            BuildActionResult result = executer.execute(
                new ExecuteBuildAction(startParameter),
                buildActionParameters,
                new DefaultBuildRequestContext(new DefaultBuildRequestMetaData(clientMetaData, startTime, sharedServices.get(ConsoleDetector.class).isConsoleInput()), new DefaultBuildCancellationToken(), new NoOpBuildEventConsumer())
            );
            if (result.hasFailure()) {
                // Don't need to unpack the serialized failure. It will already have been reported and is not used by anything downstream of this action.
                throw new ReportedException();
            }
        } finally {
            if (stoppable != null) {
                stoppable.stop();
            }
        }
    }
```

这个run实例一个executer

回头看看executer

```java
private Runnable runBuildAndCloseServices(StartParameterInternal startParameter, DaemonParameters daemonParameters, BuildActionExecuter<BuildActionParameters, BuildRequestContext> executer, ServiceRegistry sharedServices, Object... stopBeforeSharedServices) {
        BuildActionParameters parameters = createBuildActionParameters(startParameter, daemonParameters);
        Stoppable stoppable = new CompositeStoppable().add(stopBeforeSharedServices).add(sharedServices);
        return new RunBuildAction(executer, startParameter, clientMetaData(), getBuildStartTime(), parameters, sharedServices, stoppable);
    }
```

实际上就是DaemonClient

```java
private Runnable runBuildWithDaemon(StartParameterInternal startParameter, DaemonParameters daemonParameters) {
        // Create a client that will match based on the daemon startup parameters.
        ServiceRegistry clientSharedServices = createGlobalClientServices(true);
        ServiceRegistry clientServices = clientSharedServices.get(DaemonClientFactory.class).createBuildClientServices(loggingServices.get(OutputEventListener.class), daemonParameters, System.in);
        DaemonClient client = clientServices.get(DaemonClient.class);
        return runBuildAndCloseServices(startParameter, daemonParameters, client, clientSharedServices, clientServices);
    }
```

这个家伙的execute有些长

这是代码的注解

> Executes the given action in the daemon. The action and parameters must be serializable

> 代码的注释写的很清楚，先会尝试连接一个闲置或者可以使用的守护线程100次，没有闲置的守护线程或者发送build任务100次都没有回应，那么就重新创建一个守护线程，然后发送build任务，开启构建。

```java
public BuildActionResult execute(BuildAction action, BuildActionParameters parameters, BuildRequestContext requestContext) {
        UUID buildId = idGenerator.generateId();
        List<DaemonInitialConnectException> accumulatedExceptions = Lists.newArrayList();

        LOGGER.debug("Executing build {} in daemon client {pid={}}", buildId, processEnvironment.maybeGetPid());

        // Attempt to connect to an existing idle and compatible daemon
        int saneNumberOfAttempts = 100; //is it sane enough?
        for (int i = 1; i < saneNumberOfAttempts; i++) {
            final DaemonClientConnection connection = connector.connect(compatibilitySpec);
            // No existing, compatible daemon is available to try
            if (connection == null) {
                break;
            }
            // Compatible daemon was found, try it
            try {
                Build build = new Build(buildId, connection.getDaemon().getToken(), action, requestContext.getClient(), requestContext.getStartTime(), requestContext.isInteractive(), parameters);
                return executeBuild(build, connection, requestContext.getCancellationToken(), requestContext.getEventConsumer());
            } catch (DaemonInitialConnectException e) {
                // this exception means that we want to try again.
                LOGGER.debug("{}, Trying a different daemon...", e.getMessage());
                accumulatedExceptions.add(e);
            } finally {
                connection.stop();
            }
        }

        // No existing daemon was usable, so start a new one and try it once
        final DaemonClientConnection connection = connector.startDaemon(compatibilitySpec);
        try {
            Build build = new Build(buildId, connection.getDaemon().getToken(), action, requestContext.getClient(), requestContext.getStartTime(), requestContext.isInteractive(), parameters);
            return executeBuild(build, connection, requestContext.getCancellationToken(), requestContext.getEventConsumer());
        } catch (DaemonInitialConnectException e) {
            // This means we could not connect to the daemon we just started.  fail and don't try again
            throw new NoUsableDaemonFoundException("A new daemon was started but could not be connected to: " +
                "pid=" + connection.getDaemon() + ", address= " + connection.getDaemon().getAddress() + ". " +
                Documentation.userManual("troubleshooting", "network_connection").consultDocumentationMessage(),
                accumulatedExceptions);
        } finally {
            connection.stop();
        }
    }
```

这里分析一下daemon是如何开启的

先调用了connector.startDaemon

```java
 @Override
    public DaemonClientConnection startDaemon(ExplainingSpec<DaemonContext> constraint) {
        return doStartDaemon(constraint, false);
    }
```

然后调用了doStart

```java
private DaemonClientConnection doStartDaemon(ExplainingSpec<DaemonContext> constraint, boolean singleRun) {
        ProgressLogger progressLogger = progressLoggerFactory.newOperation(DefaultDaemonConnector.class)
            .start("Starting Gradle Daemon", "Starting Daemon");
        final DaemonStartupInfo startupInfo = daemonStarter.startDaemon(singleRun);
        LOGGER.debug("Started Gradle daemon {}", startupInfo);
        CountdownTimer timer = Time.startCountdownTimer(connectTimeout);
        try {
            do {
                DaemonClientConnection daemonConnection = connectToDaemonWithId(startupInfo, constraint);
                if (daemonConnection != null) {
                    startListener.daemonStarted(daemonConnection.getDaemon());
                    return daemonConnection;
                }
                try {
                    sleep(200L);
                } catch (InterruptedException e) {
                    throw UncheckedException.throwAsUncheckedException(e);
                }
            } while (!timer.hasExpired());
        } finally {
            progressLogger.completed();
        }

        throw new DaemonConnectionException("Timeout waiting to connect to the Gradle daemon.\n" + startupInfo.describe());
    }
```

doStartDaemon调用了`DaemonStarter`的`startDaemon`方法

```java
public DaemonStartupInfo startDaemon(boolean singleUse) {
        String daemonUid = UUID.randomUUID().toString();

        GradleInstallation gradleInstallation = CurrentGradleInstallation.get();
        ModuleRegistry registry = new DefaultModuleRegistry(gradleInstallation);
        ClassPath classpath;
        List<File> searchClassPath;
        if (gradleInstallation == null) {
            // When not running from a Gradle distro, need runtime impl for launcher plus the search path to look for other modules
            classpath = registry.getModule("gradle-launcher").getAllRequiredModulesClasspath();
            searchClassPath = registry.getAdditionalClassPath().getAsFiles();
        } else {
            // When running from a Gradle distro, only need launcher jar. The daemon can find everything from there.
            classpath = registry.getModule("gradle-launcher").getImplementationClasspath();
            searchClassPath = Collections.emptyList();
        }
        if (classpath.isEmpty()) {
            throw new IllegalStateException("Unable to construct a bootstrap classpath when starting the daemon");
        }

        versionValidator.validate(daemonParameters);

        List<String> daemonArgs = new ArrayList<String>();
        daemonArgs.addAll(getPriorityArgs(daemonParameters.getPriority()));
        daemonArgs.add(daemonParameters.getEffectiveJvm().getJavaExecutable().getAbsolutePath());

        List<String> daemonOpts = daemonParameters.getEffectiveJvmArgs();
        daemonArgs.addAll(daemonOpts);
        daemonArgs.add("-cp");
        daemonArgs.add(CollectionUtils.join(File.pathSeparator, classpath.getAsFiles()));

        if (Boolean.getBoolean("org.gradle.daemon.debug")) {
            daemonArgs.add("-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005");
        }
        LOGGER.debug("Using daemon args: {}", daemonArgs);

        daemonArgs.add(GradleDaemon.class.getName());
        // Version isn't used, except by a human looking at the output of jps.
        daemonArgs.add(GradleVersion.current().getVersion());

        // Serialize configuration to daemon via the process' stdin
        StreamByteBuffer buffer = new StreamByteBuffer();
        FlushableEncoder encoder = new KryoBackedEncoder(new EncodedStream.EncodedOutput(buffer.getOutputStream()));
        try {
            encoder.writeString(daemonParameters.getGradleUserHomeDir().getAbsolutePath());
            encoder.writeString(daemonDir.getBaseDir().getAbsolutePath());
            encoder.writeSmallInt(daemonParameters.getIdleTimeout());
            encoder.writeSmallInt(daemonParameters.getPeriodicCheckInterval());
            encoder.writeBoolean(singleUse);
            encoder.writeString(daemonUid);
            encoder.writeSmallInt(daemonParameters.getPriority().ordinal());
            encoder.writeSmallInt(daemonOpts.size());
            for (String daemonOpt : daemonOpts) {
                encoder.writeString(daemonOpt);
            }
            encoder.writeSmallInt(searchClassPath.size());
            for (File file : searchClassPath) {
                encoder.writeString(file.getAbsolutePath());
            }
            encoder.flush();
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
        InputStream stdInput = buffer.getInputStream();

        return startProcess(
            daemonArgs,
            daemonDir.getVersionedDir(),
            daemonParameters.getGradleUserHomeDir().getAbsoluteFile(),
            stdInput
        );
    }
```

然后进行了一些配置以后，开启了gradle的守护进程

注意这一行,这一行是守护进程运行的main方法位置。

```java
daemonArgs.add(GradleDaemon.class.getName());
```

点开startProcess

```java
private DaemonStartupInfo startProcess(List<String> args, File workingDir, File gradleUserHome, InputStream stdInput) {
        LOGGER.debug("Starting daemon process: workingDir = {}, daemonArgs: {}", workingDir, args);
        Timer clock = Time.startTimer();
        try {
            GFileUtils.mkdirs(workingDir);

            DaemonOutputConsumer outputConsumer = new DaemonOutputConsumer();

            // This factory should be injected but leaves non-daemon threads running when used from the tooling API client
            @SuppressWarnings("deprecation")
            DefaultExecActionFactory execActionFactory = DefaultExecActionFactory.root(gradleUserHome);
            try {
                ExecHandle handle = new DaemonExecHandleBuilder().build(args, workingDir, outputConsumer, stdInput, execActionFactory.newExec());

                handle.start();
                LOGGER.debug("Gradle daemon process is starting. Waiting for the daemon to detach...");
                handle.waitForFinish();
                LOGGER.debug("Gradle daemon process is now detached.");
            } finally {
                CompositeStoppable.stoppable(execActionFactory).stop();
            }

            return daemonGreeter.parseDaemonOutput(outputConsumer.getProcessOutput(), args);
        } catch (GradleException e) {
            throw e;
        } catch (Exception e) {
            throw new GradleException("Could not start Gradle daemon.", e);
        } finally {
            LOGGER.info("An attempt to start the daemon took {}.", clock.getElapsed());
        }
    }
```

注意看日志

之后就开启了daemon进程，所以start就是开启的进程。

```java
handle.start();
```

分析一下start方法

```java
public ExecHandle start() {
        LOGGER.info("Starting process '{}'. Working directory: {} Command: {} {}",
                displayName, directory, command, ARGUMENT_JOINER.join(arguments));
        lock.lock();
        try {
            if (!stateIn(ExecHandleState.INIT)) {
                throw new IllegalStateException(format("Cannot start process '%s' because it has already been started", displayName));
            }
            setState(ExecHandleState.STARTING);

            broadcast.getSource().beforeExecutionStarted(this);
            execHandleRunner = new ExecHandleRunner(this, new CompositeStreamsHandler(), processLauncher, executor);
            executor.execute(new CurrentBuildOperationPreservingRunnable(execHandleRunner));

            while (stateIn(ExecHandleState.STARTING)) {
                LOGGER.debug("Waiting until process started: {}.", displayName);
                try {
                    stateChanged.await();
                } catch (InterruptedException e) {
                    execHandleRunner.abortProcess();
                    throw UncheckedException.throwAsUncheckedException(e);
                }
            }

            if (execResult != null) {
                execResult.rethrowFailure();
            }

            LOGGER.info("Successfully started process '{}'", displayName);
        } finally {
            lock.unlock();
        }
        return this;
    }
```

整个start方法就创建了一个实例

```java
            execHandleRunner = new ExecHandleRunner(this, new CompositeStreamsHandler(), processLauncher, executor);
```

```java
 while (stateIn(ExecHandleState.STARTING)) {
                LOGGER.debug("Waiting until process started: {}.", displayName);
                try {
                    stateChanged.await();
                } catch (InterruptedException e) {
                    execHandleRunner.abortProcess();
                    throw UncheckedException.throwAsUncheckedException(e);
                }
            }
```

创建以后就等待状态发生改变

点击看看ExecHandler会发现有

```java
private Process process;
private final ProcessLauncher processLauncher;
```

然后点看processLauncher查看start的调用处,其实在Runnable

```java
@Override
    public void run() {
        try {
            startProcess();

            execHandle.started();

            LOGGER.debug("waiting until streams are handled...");
            streamsHandler.start();

            if (execHandle.isDaemon()) {
                streamsHandler.stop();
                detached();
            } else {
                int exitValue = process.waitFor();
                streamsHandler.stop();
                completed(exitValue);
            }
        } catch (Throwable t) {
            execHandle.failed(t);
        }
    }
```

分析ExecHandleRunner整个runnable的执行位置，被放进了线程池

```java
            executor.execute(new CurrentBuildOperationPreservingRunnable(execHandleRunner));入
```

一捋明白了开启daemon进程就是依靠向线程池里面放一个任务。

不过还有一点不明的是整个ProcessLauncher是哪来的

不难发现

```java
public ExecHandleRunner(DefaultExecHandle execHandle, StreamsHandler streamsHandler, ProcessLauncher processLauncher, Executor executor) {
        this.processLauncher = processLauncher;
        this.executor = executor;
        if (execHandle == null) {
            throw new IllegalArgumentException("execHandle == null!");
        }
        this.streamsHandler = streamsHandler;
        this.processBuilderFactory = new ProcessBuilderFactory();
        this.execHandle = execHandle;
    }
```

```java
 execHandleRunner = new ExecHandleRunner(this, new CompositeStreamsHandler(), processLauncher, executor);
```

最后发现其实是通过NativeService获取

```java
        processLauncher = NativeServices.getInstance().get(ProcessLauncher.class);
```

所以`GradleDaemon`就跑起来了

```java
public class GradleDaemon {
    public static void main(String[] args) {
        ProcessBootstrap.run("org.gradle.launcher.daemon.bootstrap.DaemonMain", args);
    }
}
```

他跑起来的第一想法就是调用

DaemonMain

```java
public void run(String[] args) {
        RecordingExecutionListener listener = new RecordingExecutionListener();
        try {
            doAction(args, listener);
        } catch (Throwable e) {
            createErrorHandler().execute(e);
            listener.onFailure(e);
        }

        Throwable failure = listener.getFailure();
        ExecutionCompleter completer = createCompleter();
        if (failure == null) {
            completer.complete();
        } else {
            completer.completeWithFailure(failure);
        }
    }
```

它又调用了doAction

```java
protected void doAction(String[] args, ExecutionListener listener) {
        ......
        ......

        LOGGER.debug("Assuming the daemon was started with following jvm opts: {}", startupOpts);

        Daemon daemon = daemonServices.get(Daemon.class);
        daemon.start();

        try {
            DaemonContext daemonContext = daemonServices.get(DaemonContext.class);
            Long pid = daemonContext.getPid();
            daemonStarted(pid, daemon.getUid(), daemon.getAddress(), daemonLog);
            DaemonExpirationStrategy expirationStrategy = daemonServices.get(MasterExpirationStrategy.class);
            daemon.stopOnExpiration(expirationStrategy, parameters.getPeriodicCheckIntervalMs());
        } finally {
            daemon.stop();
            // TODO: Stop all daemon services
            CompositeStoppable.stoppable(daemonServices.get(GradleUserHomeScopeServiceRegistry.class)).stop();
        }
    }
```

进入以后进行了海量的配置

之后获取了Daemon对象,以下是Daemon类的注解信息

> A long-lived build server that accepts commands via a communication channel.

Daemon就相当于一个长时间运行的服务器，通过一个管道和客户端进行通信

start某种程度上就是开启了这个管道

之后进行了一些必要的配置以后，就开始**等死**

```java
public void stopOnExpiration(DaemonExpirationStrategy expirationStrategy, int checkIntervalMills) {
        LOGGER.debug("stopOnExpiration() called on daemon");
        scheduleExpirationChecks(expirationStrategy, checkIntervalMills);
        awaitExpiration();
    }
```

```java
private void awaitExpiration() {
        LOGGER.debug("awaitExpiration() called on daemon");

        DaemonStateCoordinator stateCoordinator;
        lifecycleLock.lock();
        try {
            if (this.stateCoordinator == null) {
                throw new IllegalStateException("cannot await stop on daemon as it has not been started.");
            }
            stateCoordinator = this.stateCoordinator;
        } finally {
            lifecycleLock.unlock();
        }

        stateCoordinator.awaitStop();
    }
```

```java
 boolean awaitStop() {
        lock.lock();
        try {
            while (true) {
                try {
                    switch (state) {
                        case Idle:
                        case Busy:
                            LOGGER.debug("daemon is running. Sleeping until state changes.");
                            condition.await();
                            break;
                        case Canceled:
                            LOGGER.debug("cancel requested.");
                            cancelNow();
                            break;
                        case Broken:
                            throw new IllegalStateException("This daemon is in a broken state.");
                        case StopRequested:
                            LOGGER.debug("daemon stop has been requested. Sleeping until state changes.");
                            condition.await();
                            break;
                        case Stopped:
                            LOGGER.debug("daemon has stopped.");
                            return true;
                    }
                } catch (InterruptedException e) {
                    throw UncheckedException.throwAsUncheckedException(e);
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

就这样完成了gradle的启动。开启了一个Daemon一直等待客户端发送构建任务，当有任务来到的时候就将任务发送到线程池执行构建。

### 小结

- gradle构建采用的c/s架构模式，当第一次构建开始的时候，会触发开启一个daemon进程，一直在后台等待客户端的构建请求。（前提是gradle构建采用的是daemon模式，默认情况下是打开的）。

- Gradle构建中有几个比较重要的类，Daemon，DaemonClient。Daemon相当于一个服务器，DaemonClient就相当于一个客户机。每当我们的项目需要build的时候都会new一个DaemonClient向Daemon发送一个构建请求，然后Daemon就会将构建任务放入线程池内执行。最后构建完成通过socket发送请求到客户端。

# gradle构建

> Time :2022-2-23

> 前面分析了gradle的启动，光是启动还是不够，接下来分析一下是怎么开启构建的。

> 每一个构建任务都会被放入到daemon进程的线程池中，然后执行。

这里是任务开始的起点，也就是task.run（任务开始执行）

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220223125050.png)

之后呢略过一些不重要的任务调度，执行到了一个比较重要的类里面`DaemonCommandExecution`

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220223125351.png)

这个类很有意思的一点就是它的成员变量actions，他是一个链表。

也就是说它把需要执行的内容串连起来了，当需要执行的时候就把这个串里面的任务链进行缩减，然后调用execute

```java
private final LinkedList<DaemonCommandAction> actions;
```

这个执行任务链的过程比较长大概又15个任务。其中大部分就是一些配置任务，比如加log信息，配置环境变量。由于篇幅的原因那些不重要的配置任务就直接省略。

这里又一个任务非常重要也就是`ExecuteBuild`，这个任务触发了构建流程。

可以看到他是一个BuildCommandOnly的是实现类

```java
public class ExecuteBuild extends BuildCommandOnly
```

BuildCommandOnly又是DaemonCommandAction的实现类

```java
public abstract class BuildCommandOnly implements DaemonCommandAction
```

前面分析过了任务链的执行是依靠一层层的调用BuildCommandOnly.execute.

这里也没落下。它调用了doBuild方法

```java
@Override
    public void execute(DaemonCommandExecution execution) {
        Command command = execution.getCommand();
        if (!(command instanceof Build)) {
            throw new IllegalStateException(String.format("%1$s command action received a command that isn't Build (command is %2$s), this shouldn't happen", this.getClass(), command.getClass()));
        }

        doBuild(execution, (Build)command);
    }
```

ExecuteBuild.doBuild执行如下

```java
protected void doBuild(DaemonCommandExecution execution, Build build) {
        LOGGER.debug("The daemon has started executing the build.");
        LOGGER.debug("Executing build with daemon context: {}", execution.getDaemonContext());
        this.runningStats.buildStarted();
        DaemonConnectionBackedEventConsumer buildEventConsumer = new DaemonConnectionBackedEventConsumer(execution);

        try {
            BuildCancellationToken cancellationToken = execution.getDaemonStateControl().getCancellationToken();
            BuildRequestContext buildRequestContext = new DefaultBuildRequestContext(build.getBuildRequestMetaData(), cancellationToken, buildEventConsumer);
            if (!build.getAction().getStartParameter().isContinuous()) {
                buildRequestContext.getCancellationToken().addCallback(new Runnable() {
                    public void run() {
                        ExecuteBuild.LOGGER.info("The daemon has received a build cancellation request.");
                    }
                });
            }

            BuildActionResult result = this.actionExecuter.execute(build.getAction(), build.getParameters(), buildRequestContext);
            execution.setResult(result);
        } finally {
            buildEventConsumer.waitForFinish();
            this.runningStats.buildFinished();
            LOGGER.debug("The daemon has finished executing the build.");
        }

        execution.proceed();
    }
```

这里调用了一个非常重要的方法，执行构建并获取构建结果。在它以前的任务就是一些配置以及准备，

```java
 BuildActionResult result = this.actionExecuter.execute(build.getAction(), build.getParameters(), buildRequestContext);
```

actionExecuter.execute的执行逻辑和前面任务链的执行逻辑很像。

每一个executer内部都持有了另外一个executer，等待把当前的executer的任务执行完成以后会调用delegate的execute继续执行，确保任务链不断。

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220223131207.png)

这里也执行了一些没啥用的配置任务。直到执行到`BuildSessionLifecycleBuildActionExecuter`

这里又把任务委托给了BuildSessionState.run

这里又是采用的链式任务执行

```java
 Result result = context.execute(action);
```

```java
public BuildActionResult execute(BuildAction action, BuildActionParameters actionParameters, BuildRequestContext requestContext) {
        StartParameterInternal startParameter = action.getStartParameter();
        if (action.isCreateModel()) {
            startParameter.setContinuous(false);
        }

        CrossBuildSessionState crossBuildSessionState = new CrossBuildSessionState(this.globalServices, startParameter);

        BuildActionResult var7;
        try {
            BuildSessionState buildSessionState = new BuildSessionState(this.userHomeServiceRegistry, crossBuildSessionState, startParameter, requestContext, actionParameters.getInjectedPluginClasspath(), requestContext.getCancellationToken(), requestContext.getClient(), requestContext.getEventConsumer());

            try {
                var7 = (BuildActionResult)buildSessionState.run((context) -> {
                    Result result = context.execute(action);
                    PayloadSerializer payloadSerializer = (PayloadSerializer)context.getServices().get(PayloadSerializer.class);
                    if (result.getBuildFailure() == null) {
                        return BuildActionResult.of(payloadSerializer.serialize(result.getClientResult()));
                    } else {
                        return requestContext.getCancellationToken().isCancellationRequested() ? BuildActionResult.cancelled(payloadSerializer.serialize(result.getBuildFailure())) : BuildActionResult.failed(payloadSerializer.serialize(result.getClientFailure()));
                    }
                });
            } catch (Throwable var11) {
                try {
                    buildSessionState.close();
                } catch (Throwable var10) {
                    var11.addSuppressed(var10);
                }

                throw var11;
            }

            buildSessionState.close();
        } catch (Throwable var12) {
            try {
                crossBuildSessionState.close();
            } catch (Throwable var9) {
                var12.addSuppressed(var9);
            }

            throw var12;
        }

        crossBuildSessionState.close();
        return var7;
    }
```

然后又是一番惊天地泣鬼神的任务调度以后跑到了`DefaultBuildTreeLifecycleController`

```java
public class DefaultBuildTreeLifecycleController implements BuildTreeLifecycleController
```

又是一个doBuild

```java
 private <T> T doBuild(final BuildAction<T> build) {
        if (completed) {
            throw new IllegalStateException("Cannot run more than one action for this build.");
        }
        completed = true;
        // TODO:pm Move this to RunAsBuildOperationBuildActionRunner when BuildOperationWorkerRegistry scope is changed
        return workerLeaseService.withLocks(Collections.singleton(workerLeaseService.getWorkerLease()), () -> {
            List<Throwable> failures = new ArrayList<>();
            Consumer<Throwable> collector = failures::add;

            T result;
            try {
                result = build.run(buildLifecycleController, collector);
            } catch (Throwable t) {
                result = null;
                failures.add(t);
            }

            includedBuildControllers.finishBuild(collector);
            RuntimeException reportableFailure = exceptionAnalyser.transform(failures);
            buildLifecycleController.finishBuild(reportableFailure, collector);

            RuntimeException finalReportableFailure = exceptionAnalyser.transform(failures);
            if (finalReportableFailure != null) {
                throw finalReportableFailure;
            }

            return result;
        });
    }
```

之后创建了一个构建的生命周期管理器开启构建

`DefaultBuildLifecycleController`

代码有些多不贴出了，可以注意一下内部的一个枚举类

这代表了任务的执行周期

```java
private enum State {
        Created, Configure, TaskGraph, Finished;

        String getDisplayName() {
            if (TaskGraph == this) {
                return "Build";
            } else {
                return "Configure";
            }
        }
    }
```

又到了`VintageBuildModelController`的doBuildStages开始构建任务。

这个方法看来来就很贴切

```java
private void doBuildStages(Stage upTo) {
        prepareSettings();
        if (upTo == Stage.LoadSettings) {
            return;
        }
        prepareProjects();
        if (upTo == Stage.Configure) {
            return;
        }
        prepareTaskExecution();
    }
```

上述3个方法

- prepareSettings
  
  加载编译init脚本和顶层的settings.gradle文件**并执行**

- prepareProjects
  
  加载buildSrc以及build.gradle文件

- prepareTaskExecution
  
  执行子project的脚本文件，并执行对应的task

他们只要是执行构建脚本都会先编译，然后才执行。这里涉及到一个类

`DefaultScriptPluginFactory`(如果想对脚本的编译流程做了解的，可以看看apply方法)

给出的编译流程分两个步骤

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220223164737.png)

> // Pass 1, extract plugin requests and plugin repositories and execute buildscript {}, ignoring (i.e. not even compiling) anything else
> 
> 第一步只编译插件请求以及插件的repositories以及执行buildscript闭包，其余的都不编译

> // Pass 2, compile everything except buildscript {}, pluginManagement{}, and plugin requests, then run
> 
> 第二步是编译除开buildscript，pluginManagement以及插件请求的其余请求，并执行

编译方法调用以后就会调用对应的执行方法

```java
initialRunner.run(target, services);


Runnable buildScriptRunner = () -> runner.run(target, services);
```

接着就会执行对应的构建脚本(准确来说是执行构建脚本的编译产物)。

上述即是gradle启动到执行构建的流程。可见的是gradle的构建其实很大程度上难度不算太大，只是回调特别的多，分支比较多，得抓主线。

# build生命周期

> Time: 2022-2-28

> 我们知晓了gradle从启动到构建的大致过程，但是前面的所有内容其实并不算太重要，分析以后其实收获并不多。后续的内容就得好好分析了，因为它属于gradle的核心内容——build。

## 分析起点

`DefaultBuildTreeLifecycleController`

```java
public class DefaultBuildTreeLifecycleController implements BuildTreeLifecycleController
```

可以发现他是`BuildTreeLifecycleController`的实现类

而它的注解是这样的

> Controls the lifecycle of the build tree, allowing a single action to be run against the build tree
> 
> 控制构建树生命周期

除此之外还要注意`DefaultBuildTreeLifecycleController`的一个成员变量`BuildLifecycleController`,它是实际的管理者。它内部有一个枚举类。对于我们解读build流程比较重要。

```java
private static enum State {
        Created,
        Configure,
        TaskGraph,
        Finished;

        private State() {
        }

        String getDisplayName() {
            return TaskGraph == this ? "Build" : "Configure";
        }
    }
```

## fromBuildModel

> Configures the build, optionally schedules and runs any tasks specified in the {@link org.gradle.StartParameter} associated with the build, calls the given action and finally finishes up the build.
> 
> 配置buiild，选择性地计划执行我们在命令行里面申明地task，执行相应地action并最后结束构建。

### Controller

> 构建是具有一定的生命周期的，而生命周期的管理是采用的几个Controller进行管理的。

#### DefaultBuildLifecycleController

> 这个类是构建生命周期的总调度类。

##### 继承关系

> `DefaultBuildLifecycleController`是`BuildLifecycleController`的实现类。

```java
public class DefaultBuildLifecycleController implements BuildLifecycleController
```

> 接口的注释给的也很明确。
> 
> Controls the lifecycle of an individual build in the build tree.
> 
> 控制一个单独的构建的生命周期

#### DefaultBuildTreeLifecycleController

> Controls the lifecycle of the build tree, allowing a single action to be run against the build tree.
> 
> 掌控构建树的生命周期，允许单action构建树的运行。

#### VintageBuildModelController

> Transitions the model of an individual build in the build tree through its lifecycle.
> 
> 过渡构建树中的的构建的生命周期。

##### State

状态的可以分为如下的几个

- Created

- LoadSettings

- Configure

- TaskGraph

### 执行过程

> 如果让我直接硬分析我绝对是会崩溃的，单步调试能把人都给调试傻，gradle源码有很大一特色就是代码的调用栈特别的长，所以硬分析着实不行。

> 源码分析的突破口也就是全局生命周期的监听，gradle build具有生命周期，我们就直接在它对应生命周期的方法上打断点，可能在很大程度上能减轻海量调用栈的负担。

#### 生命周期回调

```groovy
gradle.beforeSettings {
    println "beforeSettings"
}

gradle.settingsEvaluated {
    println "settingsEvaluated"
}

gradle.projectsEvaluated {
    println "projectsEvaluated"
}

gradle.projectsLoaded {
    println "projectsLoaded"
}

gradle.afterProject {
    println "afterProject"
}

gradle.beforeProject {
    println "beforeProject"
}

gradle.buildFinished {
    println "buildFinished"
}

gradle.tingsEvaluated {
    println "tingsEvaluated"
}
```

#### 正文

##### fromBuildModel

就程序停止的位置来看，选取了`DefaultBuildTreeLifecycleController`的`fromBuildModel`方法

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228230103.png)

##### scheduleRequestedTasks

任务调度

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228230326.png)

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228230834.png)

##### prepareSettings

跳过一些没必要的东西就执行到了doLoadBuild

```java
void doLoadBuild() {
            BuildOperationFiringSettingsPreparer.this.delegate.prepareSettings(this.gradle);
        }
```

紧接着就是判断是否是 root的settings,然后去寻找并加载settings.gradle

```java
public void prepareSettings(GradleInternal gradle) {
        SettingsLoader settingsLoader = gradle.isRootBuild() ? this.settingsLoaderFactory.forTopLevelBuild() : this.settingsLoaderFactory.forNestedBuild();
        settingsLoader.findAndLoadSettings(gradle);
    }
```

然后先执行了init script

```java
public SettingsInternal findAndLoadSettings(GradleInternal gradle) {
        this.initScriptHandler.executeScripts(gradle);
        return this.delegate.findAndLoadSettings(gradle);
    }
```

然后又是一番惊天泣泣鬼神的调度以后开始编译执行init script

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228232241.png)

值得注意的是每一个脚本都是会被读取，然后编译两次，执行两次。（这点很重要）

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228232531.png)

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228232825.png)

###### beforeSettings

initScript执行以后就会去执行SettingsScript

准备执行settings.gradle,调用beforeSettings方法，最后引入插件

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228234113.png)

插件的引入还是和init script的一样

紧接着又是二次编译，二次执行(不贴了)

```java
 private void applySettingsScript(TextResourceScriptSource settingsScript, SettingsInternal settings) {
        ScriptPlugin configurer = this.configurerFactory.create(settingsScript, settings.getBuildscript(), settings.getClassLoaderScope(), settings.getBaseClassLoaderScope(), true);
        configurer.apply(settings);
    }
```

###### settingsEvaluated & tingsEvaluated

上述执行完成以后就会调用gradle的回调

在gradle脚本里面的settingsEvaluated闭包调用完成以后就会继续调用tingsEvaluated

```java
public SettingsInternal process(GradleInternal gradle, SettingsLocation settingsLocation, ClassLoaderScope buildRootClassLoaderScope, StartParameter startParameter) {
        SettingsInternal settings = this.delegate.process(gradle, settingsLocation, buildRootClassLoaderScope, startParameter);
        gradle.getBuildListenerBroadcaster().settingsEvaluated(settings);
        settings.preventFromFurtherMutation();
        return settings;
    }
```

之后就是确保include的project不包含buildSrc（也就是说settings.gradle管理的子project名称不可以是buildSrc，详细可见代码）

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220228235422.png)

###### includeBuild

准备开始includeBuild

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220301000445.png)

然后includeBuild又会去new一个`DefaultBuildLifecycleController`又走一套流程

只不过呢这里upTo是LoadSettings，也就是说只执行一个sttings操作。

prepareSettings又会先去执行init script然后执行settings.gradle

这样root project的prepareSettings就执行完了

```java
private void doBuildStages(VintageBuildModelController.Stage upTo) {
        this.prepareSettings();
        if (upTo != VintageBuildModelController.Stage.LoadSettings) {
            this.prepareProjects();
            if (upTo != VintageBuildModelController.Stage.Configure) {
                this.prepareTaskExecution();
            }
        }
    }
```

###### 小节

- rootProject下的prepareSettings先回调用init script然后再调用对应的settings script，然后再去执行includeBuild的prepareSettings去加载includeBuilde的settings script。

- beforeSettings触发时间在init script加载之后，在settings执行以前

- settingsEvaluated触发时间在settings执行以后

- tingsEvaluated触发时间在settingsEvaluated闭包执行以后

##### prepareProjects

```java
public void prepareProjects(GradleInternal gradle) {
        SettingsInternal settings = gradle.getSettings();
        ClassLoaderScope settingsClassLoaderScope = settings.getClassLoaderScope();
        ClassLoaderScope buildSrcClassLoaderScope = settingsClassLoaderScope.createChild("buildSrc[" + gradle.getIdentityPath() + "]");
        gradle.setBaseProjectClassLoaderScope(buildSrcClassLoaderScope);
        this.generateDependenciesAccessorsAndAssignPluginVersions(gradle.getServices(), settings, buildSrcClassLoaderScope);
        this.buildLoader.load(gradle.getSettings(), gradle);
        if (gradle.isRootBuild()) {
            this.buildStateRegistry.beforeConfigureRootBuild();
        }

        this.buildBuildSrcAndLockClassloader(gradle, buildSrcClassLoaderScope);
        this.delegate.prepareProjects(gradle);
        if (gradle.isRootBuild()) {
            this.buildStateRegistry.afterConfigureRootBuild();
        }

    }
```

###### projectsLoaded

![](https://gitee.com/False_Mask/pics/raw/master/PicsAndGifs/20220302112634.png)

紧接着就是判断该build是不是顶层的build(gradle的build过程类似于一个树的结构，顶层的向下一直扩展新的build，root build也就是最顶层的build，它的parent为null)

如果是就做一些其他的操作

执行includeBuild

```java
public void beforeConfigureRootBuild() {
        this.registerSubstitutions();
}

private void registerSubstitutions() {
        Iterator var1 = this.libraryBuilds.values().iterator();

        while(var1.hasNext()) {
            IncludedBuildState includedBuild = (IncludedBuildState)var1.next();
            this.currentlyConfiguring.add(includedBuild);
            this.dependencySubstitutionsBuilder.build(includedBuild);
            this.currentlyConfiguring.remove(includedBuild);
}
```

紧接着就又执行到了doBuildStage

```java
public GradleInternal getConfiguredModel() {
        this.doBuildStages(VintageBuildModelController.Stage.Configure);
        return this.gradle;
    }
```

然后就执行到了熟悉的代码片段

```java
public void prepareProjects(GradleInternal gradle) {
        SettingsInternal settings = gradle.getSettings();
        ClassLoaderScope settingsClassLoaderScope = settings.getClassLoaderScope();
        ClassLoaderScope buildSrcClassLoaderScope = settingsClassLoaderScope.createChild("buildSrc[" + gradle.getIdentityPath() + "]");
        gradle.setBaseProjectClassLoaderScope(buildSrcClassLoaderScope);
        this.generateDependenciesAccessorsAndAssignPluginVersions(gradle.getServices(), settings, buildSrcClassLoaderScope);
        this.buildLoader.load(gradle.getSettings(), gradle);
        if (gradle.isRootBuild()) {
            this.buildStateRegistry.beforeConfigureRootBuild();
        }

        this.buildBuildSrcAndLockClassloader(gradle, buildSrcClassLoaderScope);
        this.delegate.prepareProjects(gradle);
        if (gradle.isRootBuild()) {
            this.buildStateRegistry.afterConfigureRootBuild();
        }

    }
```

先回去执行buildSrc然后才会去prepareProjects

然后他呢又会依次去调用每一个字project的configure

最后prepareProjects执行结束

```java
 public void configureHierarchy(ProjectInternal project) {
        this.configure(project);
        Iterator var2 = project.getSubprojects().iterator();

        while(var2.hasNext()) {
            Project sub = (Project)var2.next();
            this.configure((ProjectInternal)sub);
        }

    }
```

###### 小结

当调用root build下的prepareProjects时候，先会去执行初始化，配置，执行buildSrc的内容，然后遍历所有的子project进行配置操作。

##### prepareTaskExecution

调用了doBuild

执行了所有的相应的task

```java
public <T> T fromBuildModel(boolean runTasks, Function<? super GradleInternal, T> action) {
        return this.doBuild((buildController, failureCollector) -> {
            if (runTasks) {
                buildController.scheduleRequestedTasks();
                List<Throwable> failures = new ArrayList();
                this.workExecutor.execute((throwable) -> {
                    failures.add(throwable);
                    failureCollector.accept(throwable);
                });
                if (!failures.isEmpty()) {
                    return null;
                }
            } else {
                buildController.getConfiguredBuild();
            }

            return action.apply(buildController.getGradle());
        });
    }
```

## 小结

- gradle构建是存有生命周期的，状态呢大致可分为3个初始化，配置，执行
  
  - 初始化过程主要是settings.gradle的执行以及一些init.gradle的执行
  
  - 配置主要是build.gradle的执行以及buildSrc的执行(通常的buildSrc是最先执行的)
  
  - 执行过程主要是执行我们在terminal里面定义的task

- gradle的构建是依靠`VintageBuildModelController`的`doBuildStages`调度执行。(`VintageBuildModelController`可以看成是build生命周期的管理者，除此之外还有两个与gradle build生命周期相关的类分别是`BuildLifecycleController`和`BuildTreeLifecycleController`)
  
  - doBuildStages有一个新参，upTo表明了代码要执行到哪个生命周期
  
  - 对应有4个状态Created,  LoadSettings,  Configure,  TaskGraph
  
  - Created, LoadSettings表明只执行到`prepareSettings`
  
  - Configure表明执行到`prepareProjects`
  
  - TaskGraph表明执行到`prepareTaskExecution`

- `doBuildStages其实和build的三个生命周期状态是吻合的
  
  - prepareSettings对应初始化
  
  - prepareProjects对应配置
  
  - prepareTaskExecution对应执行

- gradle的构建是从顶层开始执行的，依次调用prepareSettings，prepareProjects，prepareTaskExecution。在调用这3个方法的过程中会触发其他的controller的doBuildStages对某一个特定的模块进行构建，所以整个构建的流程其实不是完全按照模块的顺序进行执行的。执行顺序大体如下
  
  - prepareSettings会先把root模块下的settings.gradle编译执行，然后去编译执行所有includeBuild的settings.gradle。
  
  - prepareProjects先会去编译执行includeBuild的build.gradle然后走完buildSrc的全套生命周期，之后去编译执行root模块以及它的所有子模块的build.gradle
  
  - prepareTaskExecution会执行root模块以及所有子project相应的task
