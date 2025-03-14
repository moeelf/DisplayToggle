# Description
Turn ON/OFF the display of your Android phone, like scrcpy, using ADB Shell or Root.

Check **[Reddit Tasker](https://www.reddit.com/r/tasker/comments/12bcdnj/project_share_turn_display_onoff_dont_disturb/)** for Tasker users.

Termux users can continue to the next section.

# Building
You can follow these steps to build it in **[Termux](https://f-droid.org/repo/com.termux_1020.apk)**.

    yes | pkg upgrade -y

&nbsp;

    pkg install -y wget openjdk-21 dx android-tools

&nbsp;

    wget -O ~/android.jar "https://github.com/Sable/android-platforms/blob/master/android-35/android.jar?raw=true"

&nbsp;

    nano DisplayToggle.java

Now copy paste this:-

```
import android.os.Build;
import android.os.IBinder;
import java.lang.reflect.Method;
import java.lang.reflect.InvocationTargetException;

public class DisplayToggle {

	private static final Class<?> SURFACE_CONTROL_CLASS;
	private static final Class<?> DISPLAY_CONTROL_CLASS;

	static {
		try {
			SURFACE_CONTROL_CLASS = Class.forName("android.view.SurfaceControl");
			if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
				Class<?> classLoaderFactoryClass = Class.forName("com.android.internal.os.ClassLoaderFactory");
				Method createClassLoaderMethod = classLoaderFactoryClass.getDeclaredMethod("createClassLoader",
						String.class, String.class, String.class, ClassLoader.class, int.class, boolean.class, String.class);
				ClassLoader classLoader = (ClassLoader) createClassLoaderMethod.invoke(null,
						"/system/framework/services.jar", null, null, ClassLoader.getSystemClassLoader(), 0, true, null);
				DISPLAY_CONTROL_CLASS = classLoader.loadClass("com.android.server.display.DisplayControl");

				Method loadLibraryMethod = Runtime.class.getDeclaredMethod("loadLibrary0", Class.class, String.class);
				loadLibraryMethod.setAccessible(true);
				loadLibraryMethod.invoke(Runtime.getRuntime(), DISPLAY_CONTROL_CLASS, "android_servers");
			} else {
				DISPLAY_CONTROL_CLASS = null;
			}
		} catch (Exception e) {
			throw new AssertionError(e);
		}
	}

	public static void main(String... args) {
		System.out.println("Display mode: " + args[0]);
		int mode = Integer.parseInt(args[0]);

		try {
			if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE && DISPLAY_CONTROL_CLASS != null) {
				Method getPhysicalDisplayIdsMethod = DISPLAY_CONTROL_CLASS.getMethod("getPhysicalDisplayIds");
				Method getPhysicalDisplayTokenMethod = DISPLAY_CONTROL_CLASS.getMethod("getPhysicalDisplayToken", long.class);

				long[] displayIds = (long[]) getPhysicalDisplayIdsMethod.invoke(null);
				if (displayIds != null) {
					for (long displayId : displayIds) {
						IBinder token = (IBinder) getPhysicalDisplayTokenMethod.invoke(null, displayId);
						setDisplayPowerMode(token, mode);
					}
				}
			} else {
				setDisplayPowerMode(getBuiltInDisplay(), mode);
			}
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			System.exit(0);
		}
	}

	private static IBinder getBuiltInDisplay() throws Exception {
		Method method;
		if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {
			method = SURFACE_CONTROL_CLASS.getMethod("getBuiltInDisplay", int.class);
			return (IBinder) method.invoke(null, 0);
		} else {
			method = SURFACE_CONTROL_CLASS.getMethod("getInternalDisplayToken");
			return (IBinder) method.invoke(null);
		}
	}

	private static void setDisplayPowerMode(IBinder displayToken, int mode) throws Exception {
		Method method = SURFACE_CONTROL_CLASS.getMethod("setDisplayPowerMode", IBinder.class, int.class);
		method.invoke(null, displayToken, mode);
	}
}
```

Then press **CTRL+X+Y+ENTER** and save it.

And then create its Java CLASS file:-

    javac -Xlint:none -source 1.8 -target 1.8 -cp ~/android.jar DisplayToggle.java

Then DEX it using:-

    dx --dex --output DisplayToggle.dex DisplayToggle.class

You have compiled the DisplayToggle.dex file (around 2.5kb).

Now copy it to internal storage to use it later.

    termux-setup-storage

&nbsp;

    cp -f DisplayToggle.dex /storage/emulated/0

# How To Use It

In an ADB Shell (from Termux or PC):-

#### To Turn  Display OFF

    adb shell CLASSPATH=/storage/emulated/0/DisplayToggle.dex app_process / DisplayToggle 0

#### To Turn  Display ON

    adb shell CLASSPATH=/storage/emulated/0/DisplayToggle.dex app_process / DisplayToggle 2

#### Note:-

In Termux if you are rooted, you can just replace every `adb shell` with `su -c`.

# Credits

**[rom1v](https://blog.rom1v.com/2018/03/introducing-scrcpy/#run-a-java-main-on-android) - Method to make java code executable**

**[CheerfulPianissimo](https://github.com/Genymobile/scrcpy/issues/2888#issuecomment-1452140829) - Base Java code to make this possible**
