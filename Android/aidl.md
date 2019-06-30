# AIDL跨进程通信

## 什么是AIDL

AIDL(Android Interface Definition Language)称为Android接口定义语言，如它名字所言，它只是用来定义接口的，语法和Java类似，谷歌开发出这种语言的目的是希望简化开发者开发跨进程通信程序的难度，当我们新建了一个AIDL文件后，会在gen目录下自动生成对应的java代码。

跨进程通信在Android应用开发中使用的并不算的多，但它却是每个Android开发者必须要了解熟悉的东西，因为它涉及到Android特有的一种跨进程机制-Binder机制。如果熟悉Android Framework源码的同学一定会发现，Binder在Android源码中几乎随处可见，即使是简单的Activity跳转最后也用到了Binder机制。

## Example

今天来模拟一个场景，如果你的应用中使用了很多h5页面，那我们可以选择把WebView这个Activity拆出为一个单独的进程，这样做的好处就是两个进程相互独立，WebView的Activity发生内存泄漏甚至崩溃，都不会对主进程造成影响。但是这样做也有一个缺点，就是数据同步问题，因为多进程中单例是失效的，文件共享也要控制好读写冲突。这种情况下使用AIDL跨进程其实是一个很好的解决方案，WebView进程不关心数据是存放在内存还是文件的，需要数据的时候就向主进程请求就可以了，这样保持了数据源的单一性，也不用考虑读写冲突的问题。

1. 在src目录下新建一个aidl文件夹用于放置aidl文件，默认是在src目录下，当然也可以通过更改gradle文件更改这个目录。如果是使用Android Studio的话也可以右键->new->AIDL->AIDL file的方式来新建aidl文件。
![新建AIDL](./assets/as_new_aidl.png)
新建的文件名字为IUserManager, Android Studio会自动为我们生成一个方法，主要是给我们介绍如何使用的。

    ```java
    // IUserManager.aidl
    package com.hao.aidltest;

    // Declare any non-default types here with import statements

    interface IUserManager {
        /**
        * Demonstrates some basic types that you can use as parameters
        * and return values in AIDL.
        */
        void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
                double aDouble, String aString);
    }
    ```

    在继续开始前，我们先介绍下AIDL支持的数据类型吧

   - Java中原有的基本类型如int, long, char, boolean等
   - String类型
   - CharSequence类型
   - Parcelable类型
   - List类型，List中的所有元素都必须是以上支持的类型
   - Map类型，Map中的所有的元素都必须是以上支持的类型

2. 新建一个用户信息类UserInfo

    ```java
    public class UserInfo {
        private long id;
        private String username;
        private String password;

        public UserInfo(long id, String username, String password) {
            this.id = id;
            this.username = username;
            this.password = password;
        }

        @NonNull
        @Override
        public String toString() {
            return "userId=" + id + "\n"
                   + "username=" + username + "\n"
                   + "password=" + password;
        }
    }
   ```

3. 从上面AIDL支持的类型我们知道，如果想要想通过AIDL跨进程传递对象数据，则必须要该对象实现Parcelable接口

   ```java
   public class UserInfo implements Parcelable {
        private long id;
        private String username;
        private String password;

        public UserInfo(long id, String username, String password) {
            this.id = id;
            this.username = username;
            this.password = password;
        }

        protected UserInfo(Parcel in) {
            id = in.readLong();
            username = in.readString();
            password = in.readString();
        }

        public static final Creator<UserInfo> CREATOR = new Creator<UserInfo>() {
            @Override
            public UserInfo createFromParcel(Parcel in) {
                return new UserInfo(in);
            }

            @Override
            public UserInfo[] newArray(int size) {
                return new UserInfo[size];
            }
        };

        @NonNull
        @Override
        public String toString() {
            return "userId=" + id + "\n"
                   + "username=" + username + "\n"
                   + "password=" + password;
        }

        @Override
        public int describeContents() {
            return 0;
        }

        @Override
        public void writeToParcel(Parcel dest, int flags) {
            dest.writeLong(id);
            dest.writeString(username);
            dest.writeString(password);
        }
    }
   ```

4. 仅仅是实现Parcelable还是不够的，还需要新建一个同名文件UserInfo.aidl

   ```java
    package com.hao.aidltest;
    // 只有在这里声明了UserInfo，在IUserManager中才可以使用
    parcelable UserInfo;
   ```

5. 完成IUserManager中接口的定义

   ```java
    package com.hao.aidltest;

    import com.hao.aidltest.UserInfo;

    interface IUserManager {
        // 获取用户信息
        UserInfo getUserInfo();
        // 更新用户信息
        void updateUserInfo(in UserInfo info);
    }
   ```

   这里有一点需要注意，尽管UserInfo和IUserManager是在同一个package下的，但仍然需要手动导入UserInfo。这一点和Java有些区别。

6. 上面是如何定义AIDL接口，下面我们要做的是实现接口，首先我们要新建一个Service

   ```java
   public class UserInfoService extends Service {
        @Nullable
        @Override
        public IBinder onBind(Intent intent) {
        return null;
        }
    }
   ```

7. 新建一个IUserManager.Stub对象并在UserInfoService的onBind方法中返回，新建之前记得rebuild一下工程，否则可能IUserManager.Stub里面的方法没有及时的生成同步。

   ```java
   public class UserInfoService extends Service {
        private Binder binder = new IUserManager.Stub() {
            @Override
            public UserInfo getUserInfo() throws RemoteException {
                return UserManager.INSTANCE.getUserInfo();
            }

            @Override
            public void updateUserInfo(UserInfo info) throws RemoteException {
                UserManager.INSTANCE.updateUserInfo(info);
            }
        };

        @Nullable
        @Override
        public IBinder onBind(Intent intent) {
            return binder;
        }
    }
   ```

   上面那两个方法就是我们在AIDL中定义的接口，这里才是真正的实现接口的地方。为了实现那两个方法，我们用一个单例类来管理应用中的用户信息

   ```java
    public enum  UserManager {
        INSTANCE;

        private UserInfo userInfo;

        public UserInfo getUserInfo() {
            return userInfo;
        }

        public void updateUserInfo(UserInfo userInfo) {
            this.userInfo = userInfo;
        }
    }
   ```

8. 接下来就是如何使用了，首先我们需要一个运行在子进程的WebViewActivity去获取服务

   ```java
    public class WebViewActivity extends Activity {
        private IUserManager userManager;

        @Override
        protected void onCreate(@Nullable Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_webview);
            bindService(new Intent(this, UserService.class), new ServiceConnection() {
                @Override
                public void onServiceConnected(ComponentName name, IBinder service) {
                    userManager = IUserManager.Stub.asInterface(service);
                }

                @Override
                public void onServiceDisconnected(ComponentName name) {
                    userManager = null;
                }
            }, Context.BIND_AUTO_CREATE);
        }
    }
   ```

   在onServiceConnected方法中通过IUserManager.Stub.asInterface就可以获取到IUserManager的实例，然后就可以使用IUserManager中的方法了。

## 注意事项

1. 子进程通过bindService获取服务的过程是耗时的，也就是说如果Activity的onCreate或者第一次onResume使用时，IUserManager很可能还未初始化完成，所以在使用前记得判空，或者把逻辑移动到onServiceConnected方法中
2. 如果WebViewActivity在调用IUserManager的updateUserInfo方法是在子线程调用的，那么主进程的Service中updateUserInfo也会是在子进程调用
3. 通过IUserManager调用的方式都是同步的，所以如果服务执行的时间很长，很可能会造成ANR
