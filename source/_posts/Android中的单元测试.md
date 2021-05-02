---
title: Android中的单元测试
tags: [Android]
date: 2021-04-25 10:18:51
keywords: [Android,unit test,TDD,测试驱动开发,Junit,Mockito,Robolectric]
---



#### 纯java代码的单元测试

这里的纯java代码指的是不包含Android包中的代码，我们使用Junit写单元测试即可。

比如我们有一个方法是用来格式化数字，返回保留两位小数后的字符串，方法如下

``` java
public static String numberFormat(double number){
  return String.format(Locale.getDefault(),"%.2f",number);
}
```

那么我们的单元测试可以这么写，依赖一下junit测试框架`testImplementation 'junit:junit:4.+'`

``` java
import org.junit.Test;
import static org.junit.Assert.*;

public class ExampleUnitTest {
    @Test
    public void testNumberFormat(){
        assertEquals("0.23",Util.numberFormat(0.232323));
    }
}
```

<!--more-->

这里说明一下，我们在写单元测试的时候，经常会需要初始化一些数据，但我们又不想在每个测试方法中都调用一遍初始化的方法，这里测试框架给出了四个注解

``` java
@BeforeClass
public static void init() {

}
@Before
public void setUp(){

}
@After
public void tearDown(){

}
@AfterClass
public static void destroy(){

}
```

* @BeforeClass:只会执行一次，修饰的方法必须是静态的
* @AfterClass：只会执行一次，修饰的方法必须是静态的
* @Before：每次调用测试方法时都会执行一次
* @After：每次测试方法执行完成后都会执行一次



#### 代码中包含Android代码

但是，假如我们的代码中不小心"混入"了一些调用Android包的功能，比如验证邮箱的有效性，代码可能是这样的

``` java
import android.util.Patterns;
public static boolean isEmailAddress(String address){
  return Patterns.EMAIL_ADDRESS.matcher(address).matches();
}
```

这里导入了 android.util包，如果使用junit的话，在单元测试代码中会报一个空指针异常

``` java
@Test
public void testEmailAddress(){
  Assert.assertTrue(Util.isEmailAddress("gg@gg.com"));
  Assert.assertTrue(Util.isEmailAddress("huangyuan@chunyu.me"));
  Assert.assertFalse(Util.isEmailAddress("wwww"));
}
```

``` verilog
java.lang.NullPointerException
	at com.huangyuanlove.tdd_demo.Util.isEmailAddress(Util.java:15)
	at com.huangyuanlove.tdd_demo.ExampleUnitTest.testEmailAddress(ExampleUnitTest.java:27)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:59)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:56)
	at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
	at org.junit.runners.ParentRunner$3.evaluate(ParentRunner.java:306)
	at org.junit.runners.BlockJUnit4ClassRunner$1.evaluate(BlockJUnit4ClassRunner.java:100)
	at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:366)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:103)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:63)
	at org.junit.runners.ParentRunner$4.run(ParentRunner.java:331)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:79)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:329)
	at org.junit.runners.ParentRunner.access$100(ParentRunner.java:66)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:293)
	at org.junit.runners.ParentRunner$3.evaluate(ParentRunner.java:306)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:413)
	at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
	at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:68)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:33)
	at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:230)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:58)
```

因为我们的junit是跑在jvm上的，默认并没有加载android的包，这时候我们可以使用[Robolectric](http://robolectric.org/)这个三方包来做测试。在gradle中添加一下依赖`testImplementation 'org.robolectric:robolectric:4.2'`,**别问为啥不用4.4,因为我还没整明白，Shadows方法不能用**

在我们的单元测试类上加下注解，如下

``` java
@RunWith(RobolectricTestRunner.class)
public class UtilTestWithRobolectric {
    @Test
    public void testEmailAddress(){
        Assert.assertTrue(Util.isEmailAddress("gg@gg.com"));
        Assert.assertTrue(Util.isEmailAddress("huangyuan@chunyu.me"));
        Assert.assertFalse(Util.isEmailAddress("wwww"));
    }
}
```

除了这个，我们还可以使用Robolectric来测试一些页面行为

``` java
import androidx.test.core.app.ActivityScenario;
import androidx.test.core.app.ApplicationProvider;
import org.junit.Assert;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.robolectric.RobolectricTestRunner;
import org.robolectric.Shadows;
import org.robolectric.shadows.ShadowAlertDialog;
import org.robolectric.shadows.ShadowToast;
@RunWith(RobolectricTestRunner.class)
public class UtilTestWithRobolectric {
    private ActivityScenario<MainActivity> scenario;

    @Before
    public void initScenario(){
         scenario = ActivityScenario.launch(MainActivity.class);
    }
    @Test
    public void testScenario(){
        Assert.assertNotNull(scenario);
    }

    @Test
    public void testEmailAddress(){
        Assert.assertTrue(Util.isEmailAddress("gg@gg.com"));
        Assert.assertTrue(Util.isEmailAddress("huangyuan@chunyu.me"));
        Assert.assertFalse(Util.isEmailAddress("wwww"));
    }

    @Test
    //是否弹出toast
    public void testShowToast(){
        scenario.onActivity(new ActivityScenario.ActivityAction<MainActivity>() {
            @Override
            public void perform(MainActivity activity) {
                Toast toast = ShadowToast.getLatestToast();
                Assert.assertNull(toast);

                activity.findViewById(R.id.show_toast).performClick();
                toast = ShadowToast.getLatestToast();
                Assert.assertNotNull(toast);

                ShadowToast shadowToast = Shadows.shadowOf(toast);
                Assert.assertEquals("show_toast",ShadowToast.getTextOfLatestToast());
                Assert.assertEquals(Toast.LENGTH_SHORT,toast.getDuration());

            }
        });
    }

    @Test
  	//是否展示Dialog
    public void testShowDialog(){
        scenario.onActivity(new ActivityScenario.ActivityAction<MainActivity>() {
            @Override
            public void perform(MainActivity activity) {
                AlertDialog dialog = ShadowAlertDialog.getLatestAlertDialog();
                Assert.assertNull(dialog);
                activity.findViewById(R.id.show_toast).performClick();

                dialog = ShadowAlertDialog.getLatestAlertDialog();
                Assert.assertNotNull(dialog);

                ShadowAlertDialog shadowDialog = Shadows.shadowOf(dialog);
                Assert.assertEquals("Hello！", shadowDialog.getMessage());

            }
        });

    }


    @Test
  	//是否跳转到了指定页面
    public void testGoToLogin(){

        scenario.onActivity(new ActivityScenario.ActivityAction<MainActivity>() {
            @Override
            public void perform(MainActivity activity) {
                activity.findViewById(R.id.login).performClick();
                Intent expectedIntent = new Intent(activity, LoginActivity.class);
                Intent actual = Shadows.shadowOf(activity).getNextStartedActivity();
                Assert.assertEquals(expectedIntent.getComponent(),actual.getComponent());
            }
        });

    }

}

```

这里的`ActivityScenario`是使用的 androidx.test.core.app包下的类，需要依赖`testImplementation 'androidx.test:core:1.1.0'`

当然在`Robolectric`中也有对应的创建Activity的方法，不过在4.4版本中被废弃了，也推荐使用androidx.test包中创建Activity的方法。

#### Mock和Mockito
如何测试一个没有返回值的方法，一般是来看这个方法有没有得到调用。
假如我们有如下代码

``` java
public class LoginPresenter {
   public void setUserManager(UserManager userManager) {
        this.userManager = userManager;
    }
	  public void login(String userName,String password){
        userManager.performLogin(userName,password);
    }
}
```

我们要验证`mUserManager`的一些行为，首先要 mock UserManager 这个类，mock 这个类的方式是：
`Mockito.mock(UserManager.class);`
mock 了`UserManager`类之后，我们就可以开始测试了,验证一个对象的方法调用情况的方法是：
`Mockito.verify(objectToVerify).methodToVerify(arguments);`
其中，`objectToVerify`和`methodToVerify`分别是你想要验证的对象和方法

``` java
@Test
public void testLogin() {
  UserManager userManager = Mockito.mock(UserManager.class);
  LoginPresenter loginPresenter = new LoginPresenter();
  loginPresenter.setUserManager(userManager);
  loginPresenter.login("a", "b");
  Mockito.verify(userManager).performLogin("a", "b");
}
```

再假如我们在登录的时候需要先验证密码强度，但是我们测试的时候不关心这个验证逻辑，希望不管传入的密码是啥，都可以通过验证。我们就需要干预某些mock对象的方法行为

``` java
public void loginWithVerifyPassword(PasswordValidator passwordValidator, String userName,String password){
  if(passwordValidator.verifyPassword(password)){
    userManager.performLogin(userName,password);
  }else{
    System.out.println("密码不正确");
  }
}

public static class PasswordValidator{
  public boolean verifyPassword(String password){
    return password != null && password.length() >5;
  }
}
```

我们需要mock一下PasswordValidator这个类中verifyPassword的行为,这种指定 mock 对象的某个方法，让它返回特定值的写法如下：
`Mockito.when(mockObject.targetMethod(args)).thenReturn(desiredReturnValue);`代码如下

``` java
@Test
public void testLoginWithPasswordValidator() {
  LoginPresenter.PasswordValidator passwordValidator = Mockito.mock(LoginPresenter.PasswordValidator.class);
  //验证方法行为是否被改变
  Mockito.when(passwordValidator.verifyPassword(ArgumentMatchers.any())).thenReturn(true);
  Assert.assertTrue(passwordValidator.verifyPassword(""));
  Assert.assertTrue(passwordValidator.verifyPassword(null));
  Assert.assertTrue(passwordValidator.verifyPassword("123"));
  Assert.assertTrue(passwordValidator.verifyPassword("321321321"));

  //验证登录方法是否被调用
  UserManager userManager = Mockito.mock(UserManager.class);
  LoginPresenter loginPresenter = new LoginPresenter();
  loginPresenter.setUserManager(userManager);
  loginPresenter.loginWithVerifyPassword(passwordValidator, "a", "b");
  Mockito.verify(userManager).performLogin("a", "b");


}
```

#### Androidx.test

最近在看Androidx包下的测试框架，对于我们来讲，单元测试不是很多，测试代码跑在模拟器或者真机上带来的时间消耗还是可以接受的。有时间撸一下对应的代码



----

以上