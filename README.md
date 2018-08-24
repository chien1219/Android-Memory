# Android-Memory
**Richard Chien**  
Details in my Android memory usage research  
I will introduce from simple to hard, from some usage to details  
A know how from first month of my interns in 2017.  

## Several adb commands to log memory info
<pre>
● adb shell cat /proc/meminfo			    RAM info
● adb shell dumpsys meminfo <pid>	                    Process info
● adb shell procrank				    Process info(Java heap only)
● adb shell showmap <pid>		                    Very detailed
</pre>

## Tools in Android Studio
* MAT
* DDMS  
  
 As we all know the Android Framework below, We will focus on the **Android Runtime** part so as to help analyze memory in our app development.  
![Android Framework](https://i1.read01.com/SIG=gh55qt/3044325433323030.jpg)  
  
## Address space in RAM  
![Run stack](https://i.imgur.com/n3TwaAm.png)  
  
  ***
  
### /proc/meminfo  
Let's make a quick look at some of columns  
  
**HightTotal**：The total sum of physical RAM of address > 860M , can only be used by user.  
  
**HightFree**：The unused of physical RAM of address > 860M 
  
**LowTotal**：The sum of RAM which could be used by Kernel, User in RAM  
For a 512M RAM, lowTotal = MemTotal    
  
**LowFree** : the unused by whether kernel or user RAM  
lowFree = MemFree in a 512M RAM  
  
  ***
  
### Dumpsys
The following image shows the **dumpsys** memory information from an global launched app (D.A.U. about 3M).   
  
![Dumpsys](https://i.imgur.com/hG0XA4m.jpg)  
  
To illustrate different meaning in each blocks [Click me for more detail](https://www.jianshu.com/p/e48ffb4738e1)  
  
**Dalvik Heap**：Dalvik allocated RAM of your app  
  
**Pss Total** includes all allocated memory from **Zygote** , will be discussed later.  
  
**Private Dirty** : the most important part of an runtime app because it is only used by your own process. It is only stored in RAM, which means that it cant be page into storage because Android doesn't support swap.  
  
**Heap Alloc** is the sum of Dalvik and native heap.  
This value is greater than PSS total and Private Dirty because process is forked from Zygote, and allowed to share with others.  
  
**Private Clean** means the ''clean'' memory generated from your code.  
  
Next, let's have a quick comparison between **PSS** and it's family.  
  
![PSS](https://i.imgur.com/z2cvbrl.jpg)  
  
We can conclude from the above picture  
PSS is a very useful number because when the PSS for all processes in the system are summed together, that is a good representation for the total memory usage in the system. When a process is killed, the shared libraries that contributed to its PSS will be proportionally distributed to the PSS totals for the remaining processes still using that library. In this way PSS can be slightly misleading, because when a process is killed, PSS does not accurately represent the memory returned to the overall system.  
  
Another more specific example here:  
We have **20MB** for Private  
and **12 MB** Shared (Between 4 processes)  
**PSS** = 20 + (12/4) = 23  
  
  ***
  
### Procrank
And Now we start to look at **procrank** 
![Total](https://i.imgur.com/U5CfH5P.png)  
  
***
  
## Ways to use in your project  
  
There are three API within Java to access memory info.
* [Debug](https://developer.android.com/reference/android/os/Debug.MemoryInfo)
* [Activity Manager](https://developer.android.com/reference/android/app/ActivityManager.MemoryInfo)
* [Runtime (Java Native)](https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#totalMemory--)
  
We will not discuss **Runtime** class because it's different from two others  
Despite fast speed in running this API (about 1000 times faster than two others)  
but its value differ and fluctuate a lot, which results in hardly precision.
  
### Debug  vs. ActivityManager
Ways to show memory by Activity Manager:  
` void getMemoryInfo(ActivityManager.MemoryInfo outInfo)`  
` Debug.MemoryInfo getProcessMemoryInfo(int[ ] pids)`  
` List<ActivityManager.RunningAppProcessInfo>getRunningAppProcess()`  
` List<ActivityManager.RunningTaskInfoo>getRunningTasks(int maxNum)`  
` Void killBackgroundProcess(String packageName)`  
  
Detailed implementation:
  
```
private static final class ActivityManagerMemoryGetter implements MemoryGetter {
    private final ActivityManager activityManager = (ActivityManager) Globals.getInstance().getSystemService(Context.ACTIVITY_SERVICE);
    private final int[] pids = new int[]{Process.myPid()};

    private Debug.MemoryInfo info;

    @Override
    public void refresh() {
        Debug.MemoryInfo[] infos = activityManager.getProcessMemoryInfo(pids);
        info = infos[0];
    }

    @Override
    public long getDalvikPrivateDirty() {
        return info.dalvikPrivateDirty;
    }
}
private static final class DebugMemoryGetter implements MemoryGetter {
   private Debug.MemoryInfo info;

   @Override
   public void refresh() {
       info = new Debug.MemoryInfo();
       Debug.getMemoryInfo(info);
   }

   @Override
   public long getDalvikPrivateDirty() {
       return info.dalvikPrivateDirty;
   }
}
 ```
   
## Performance Comparison
  
** Speed Comparison**
  
The following image shows the time taken to finish two memory getter above.  
we can see that **ActivityManager** is slightly slower than **Debug**
![com_time](https://i.imgur.com/txI9Uco.png)  
  
  
Next, we compare the output memory usage value differ.  
we can see that both API are used to have same value when APP is running with non-dramatic burst  
(simple click on an button without page or context change)  
However, when there's a dramatic burst on memory (for method like start camera, activity...)  
the value gets different.  
![com_mem](https://i.imgur.com/Ph3zhWq.png)  
  
  
Hence as a conclusion, there two API are used in several auto-run memory checker  
Determining which to use totally depends on demand.  
some argue that **Activity manager** is developed better and has a better performance  
but  **Debug** seems to hit the bull's-eye.
