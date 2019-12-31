* ThreadLocal是一个线程本地变量。
* ThreadLocal里面有个内部类，ThreadLocalMap,用来存储线程副本。ThreadLocalMap里面的数据结构是一个Entry<k，v>一个弱引用。
* 比如有一个共享变量，private static final ThreadLocal t = new ThreadLocal();
* 线程1进来，存放一个变量t.set(object1);线程2也是用这个共享变量t，存放一个变量t.set(object2);线程1在获取t的变量时t.get()会返回obejct1,线程2在获取t的变量时t.get()会返回obejct2。
* ThreadLocal除了可以通过set(object1)来存放线程副本，也可以通过实现initialValue方法，将initialValue方法的返回值存入ThreadLocalMap里面。例如：
  ```
  Java7中的SimpleDateFormat不是线程安全的，可以用ThreadLocal来解决这个问题：
    public class DateUtil {
        private static ThreadLocal<SimpleDateFormat> format1 = new ThreadLocal<SimpleDateFormat>() {
            @Override
            protected SimpleDateFormat initialValue() {
                return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            }
        };

        public static String formatDate(Date date) {
            return format1.get().format(date);
        }
    }

  ```