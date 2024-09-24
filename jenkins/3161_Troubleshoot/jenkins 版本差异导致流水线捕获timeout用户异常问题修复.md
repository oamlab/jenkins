### 背景

**1.需求**
在jenkins pipeline中实现使用`Input` 带有`timeout`的输入步骤，并在达到超时时继续执行（有默认值）流水线。
例如以下场景：在代码提交后，自动打包完成并部署后，可以同时执行单元测试和回归测试，但回归测试可以人工干预取消，如果没有执行取消操作，则默认会进行回归测试。

**2.实现方式**
在jenkins 2.421版本之前的实现方式参考如下代码：
```
def userInput = true
def didTimeout = false
try {
    timeout(time: 15, unit: 'SECONDS') { // change to a convenient timeout for you
        userInput = input(
        id: 'Proceed1', message: 'Was this successful?', parameters: [
        [$class: 'BooleanParameterDefinition', defaultValue: true, description: '', name: 'Please confirm you agree with this']
        ])
    }
} catch(err) { // timeout reached or input false
    def user = err.getCauses()[0].getUser()
    if('SYSTEM' == user.toString()) { // SYSTEM means timeout.
        didTimeout = true
    } else {
        userInput = false
        echo "Aborted by: [${user}]"
    }
}

node {
    if (didTimeout) {
        // do something on timeout
        echo "no input was received before timeout"
    } else if (userInput == true) {
        // do something
        echo "this was successful"
    } else {
        // do something else
        echo "this was not successful"
        currentBuild.result = 'FAILURE'
    } 
}
```

**3.异常**
升级jenkins版本到2.747版本后，执行上面的pipeline，如果在input`Input requested`输入信息时，点击
`Proceed`或者`Abort`都可以正常执行，并且在点击`Abort`后会显示信息为`Aborted by: [xxx]`用户信息，但如果等待timeout后`Cancelling nested steps due to timeout`，则会抛出异常信息如下：
```
Also:   hudson.remoting.ProxyException: org.jenkinsci.plugins.workflow.actions.ErrorAction$ErrorId: 45d013d9-f6ea-41e7-8bf9-37672cba2b67
hudson.remoting.ProxyException: groovy.lang.MissingMethodException: No signature of method: org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution$ExceededTimeout.getUser() is applicable for argument types: () values: []
Possible solutions: getAt(java.lang.String), use([Ljava.lang.Object;), use(java.util.List, groovy.lang.Closure), use(java.lang.Class, groovy.lang.Closure), getClass(), getNodeId()
	at PluginClassLoader for script-security//org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SandboxInterceptor.onMethodCall(SandboxInterceptor.java:159)
	at PluginClassLoader for script-security//org.kohsuke.groovy.sandbox.impl.Checker$1.call(Checker.java:178)
	at PluginClassLoader for script-security//org.kohsuke.groovy.sandbox.impl.Checker.checkedCall(Checker.java:182)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.sandbox.SandboxInvoker.methodCall(SandboxInvoker.java:17)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.LoggingInvoker.methodCall(LoggingInvoker.java:117)
	at WorkflowScript.run(WorkflowScript:11)
	at ___cps.transform___(Native Method)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.impl.ContinuationGroup.methodCall(ContinuationGroup.java:90)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.impl.FunctionCallBlock$ContinuationImpl.dispatchOrArg(FunctionCallBlock.java:116)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.impl.FunctionCallBlock$ContinuationImpl.fixName(FunctionCallBlock.java:80)
	at jdk.internal.reflect.GeneratedMethodAccessor284.invoke(Unknown Source)
	at java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.base/java.lang.reflect.Method.invoke(Method.java:568)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.impl.ContinuationPtr$ContinuationImpl.receive(ContinuationPtr.java:72)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.impl.ConstantBlock.eval(ConstantBlock.java:21)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.Next.step(Next.java:83)
	at PluginClassLoader for workflow-cps//com.cloudbees.groovy.cps.Continuable.run0(Continuable.java:147)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.SandboxContinuable.access$001(SandboxContinuable.java:17)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.SandboxContinuable.run0(SandboxContinuable.java:49)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsThread.runNextChunk(CpsThread.java:180)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsThreadGroup.run(CpsThreadGroup.java:423)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsThreadGroup$2.call(CpsThreadGroup.java:331)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsThreadGroup$2.call(CpsThreadGroup.java:295)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsVmExecutorService.lambda$wrap$4(CpsVmExecutorService.java:140)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at hudson.remoting.SingleLaneExecutorService$1.run(SingleLaneExecutorService.java:139)
	at jenkins.util.ContextResettingExecutorService$1.run(ContextResettingExecutorService.java:28)
	at jenkins.security.ImpersonatingExecutorService$1.run(ImpersonatingExecutorService.java:68)
	at jenkins.util.ErrorLoggingExecutorService.lambda$wrap$0(ErrorLoggingExecutorService.java:51)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsVmExecutorService$1.call(CpsVmExecutorService.java:53)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsVmExecutorService$1.call(CpsVmExecutorService.java:50)
	at org.codehaus.groovy.runtime.GroovyCategorySupport$ThreadCategoryInfo.use(GroovyCategorySupport.java:136)
	at org.codehaus.groovy.runtime.GroovyCategorySupport.use(GroovyCategorySupport.java:275)
	at PluginClassLoader for workflow-cps//org.jenkinsci.plugins.workflow.cps.CpsVmExecutorService.lambda$categoryThreadFactory$0(CpsVmExecutorService.java:50)
	at java.base/java.lang.Thread.run(Thread.java:833)
Finished: FAILURE
```
### 问题分析
**问题推断:**
报错关键字`No signature of method`,肯定是升级jenkins后的某些方法发生变化导致。

**问题验证**
修改pipeline中代码取值，并打印相关信息，代码如下
```
def userInput = true
def didTimeout = false
try {
    timeout(time: 5, unit: 'SECONDS') { // change to a convenient timeout for you
        input id: 'Continue', message: '是否继续', ok: 'OK'
    }

} 

catch(err) { // timeout reached or input false
    //def user = err.getCauses()[0].getUser() 
    def user = err.getCauses()[0].toString()
     echo "the Causes is ：" + user     
    if ('SYSTEM' == user ){ // SYSTEM means timeout. 
        didTimeout = true
    } else {
        userInput = false
        echo "Aborted by: [${user}]"
        //throw err
    }
    println("didtimeout is " + didTimeout )
    
}

node {
    if (didTimeout) {
        // do something on timeout
        echo "no input was received before timeout"
    } else if (userInput == true) {
        // do something
        echo "this was successful"
    } else {
        // do something else
        echo "this was not successful"
        currentBuild.result = 'FAILURE'
    } 
}
```
执行过程中,点击`是否继续 OK or Abort` ` Abort` 输出信息为：
```
Started by user 邱科
[Pipeline] Start of Pipeline
[Pipeline] timeout
Timeout set to expire in 5 sec
[Pipeline] {
[Pipeline] input
是否继续
OK or Abort
[Pipeline] }
[Pipeline] // timeout
[Pipeline] echo
the Causes is ：org.jenkinsci.plugins.workflow.support.steps.input.Rejection@443cd320
[Pipeline] echo
Aborted by: [org.jenkinsci.plugins.workflow.support.steps.input.Rejection@443cd320]
[Pipeline] echo
didtimeout is false
[Pipeline] node
Running on Jenkins in /data/jenkins_home/jobs/z-test-pipeline/workspace
[Pipeline] {
[Pipeline] echo
this was not successful
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: FAILURE

```
不执行任何操作，等待`timeout`输出信息为：
```
Started by user 邱科
[Pipeline] Start of Pipeline
[Pipeline] timeout
Timeout set to expire in 5 sec
[Pipeline] {
[Pipeline] input
是否继续
OK or Abort
Cancelling nested steps due to timeout
[Pipeline] }
[Pipeline] // timeout
[Pipeline] echo
the Causes is ：org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution$ExceededTimeout@490d482e
[Pipeline] echo
Aborted by: [org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution$ExceededTimeout@490d482e]
[Pipeline] echo
didtimeout is false
[Pipeline] node
Running on Jenkins in /data/jenkins_home/jobs/z-test-pipeline/workspace
[Pipeline] {
[Pipeline] echo
this was not successful
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: FAILURE
```
**Causes差异**
手动点击 `Abort` 输出为`org.jenkinsci.plugins.workflow.support.steps.input.Rejection@443cd320`
`timeout`输出为：`org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution$ExceededTimeout@490d482e`

在**jenkins 2.421**版本中执行以上代码，无论是点击`Abort`还是等待`timeout`输出的信息为：
```
Started by user admin
[Pipeline] Start of Pipeline (hide)
[Pipeline] timeout
Timeout set to expire in 15 sec
[Pipeline] {
[Pipeline] input
是否继续
OK or Abort
[Pipeline] }
[Pipeline] // timeout
[Pipeline] echo
the Causes is ：org.jenkinsci.plugins.workflow.support.steps.input.Rejection@1106f4fd
[Pipeline] echo
Aborted by: [org.jenkinsci.plugins.workflow.support.steps.input.Rejection@1106f4fd]
[Pipeline] echo
didtimeout is false
[Pipeline] node
Running on Jenkins in /root/.jenkins/workspace/z-test
[Pipeline] {
[Pipeline] echo
this was not successful
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: FAILURE
```
**Causes**在旧版本`2.421`中的方法都是相同的方法：`org.jenkinsci.plugins.workflow.support.steps.input.Rejection`，在`2.747`版本中`timeout`的方法已经修改为`org.jenkinsci.plugins.workflow.steps.TimeoutStepExecution$ExceededTimeout`

>没有去jenkins源码中详细查验哪个版本进行的修改。

### 修复问题
在新的jenkins版本中，使用`Input` 带有`timeout`的输入步骤中`Abort`和`timeout`的`Causes`发生了改变，`timeout`中没有了`.getUser()`方法，于是使用`Causes`做正则匹配来区分是`Abort`和`timeout`,并进行赋值来执行不同的操作。

代码如下：
```
def userInput = true
def didTimeout = false
try {
    timeout(time: 5, unit: 'SECONDS') { // change to a convenient timeout for you
        input id: 'Continue', message: '是否继续', ok: 'OK'
    }

} 

catch(err) { // timeout reached or input false
    //def user = err.getCauses()[0].getUser() 
    def user = err.getCauses()[0].toString()
    // echo "the Causes is ：" + user     
     
   // if ('SYSTEM' == user ){ // SYSTEM means timeout. 
   if( user =~ 'TimeoutStepExecution') { //match timeout Causes 
        didTimeout = true
    } else {
        userInput = false
        //echo "Aborted by: [${user}]"
        //throw err
    }
    println("didtimeout is " + didTimeout )
    
}

node {
    if (didTimeout) {
        // do something on timeout
        echo "no input was received before timeout"
    } else if (userInput == true) {
        // do something
        echo "this was successful"
    } else {
        // do something else
        echo "this was not successful"
        currentBuild.result = 'FAILURE'
    } 
}
```
问题解决

###  参考文档
https://support.cloudbees.com/hc/en-us/articles/226554067-Pipeline-How-to-add-an-input-step-with-timeout-that-continues-if-timeout-is-reached-using-a-default-value