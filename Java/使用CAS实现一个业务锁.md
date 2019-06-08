### 背景

最近做的项目有这样一个需求，我们有一个问题记录，每一个问题记录有一个整改人员字段，这个整改人员是可以有多个人的。整改人员可以对这个问题进行整改，但是业务要求同时只能有一个整改人可以进入整改页面，当有一个整改者在进行整改时，提示当前有另一个整改者在整改，请稍后。

### 解决方案

最开始我想使用一个全局锁来解决这个问题，但是因为用户操作的不可控行，机器的意外故障等原因可能使得用户加锁后但是没有释放锁，这样会导致发生死锁问题。于是我打算实现一个可以设置过期时间的锁，这样就能解决死锁的问题。但是这样会存在一个问题，就是首先A整改者获取锁进入整改页面，但是迟迟没有提交问题记录并且释放锁，这个时候因为锁过期了，有另外一个B整改者获取锁进入整改页面，这样就有两个整改者在整改了。但是可以在提交整改成果时结合数据库的乐观锁保证只有一个用户能够修改成功，也就是判断下此问题记录的修改时间，如果修改时间在整改期间内没有变化过，说明没有别的整改者提交这个问题，如果不相等，说明有别的整改者整改了，需要退出重新获取锁进入整改页面整改。

因为JDK提供的synchronized关键字以及Lock组件都没有锁过期的功能，于是我使用CAS实现了一个可以设置过期时间的锁。该锁必须满足以下几点:
* **同一时刻只能有一个用户获取锁(锁是有效的, 没有过期)。**

* **锁需要有一个过期时间, 避免发生死锁。**

* **只有拥有锁的用户才能释放锁。**

本来想画个流程图的，但是代码其实也很简单，看下就懂了

```java
public class BusinessLock {

    private static class LockObj {

        /**
         * 拥有锁的用户
         */
        String owner;

        /**
         * 锁的过期时间
         */
        long expireTime;

        LockObj(String owner, long expireTime) {
            this.owner = owner;
            this.expireTime = expireTime;
        }
    }

    private AtomicReference<LockObj> lockObjReference = new AtomicReference<>();

    /**
     * 尝试获取锁
     * @param owner 锁的标识
     * @param expireTime 锁的过期时间(单位为秒)
     * @return 获取锁成功返回true, 否则返回false
     */
    public boolean tryLock(String owner, long expireTime) {
        LockObj lockObj = new LockObj(owner, System.currentTimeMillis() + 
                                      expireTime * 1000);
        // 获取锁成功
        if (lockObjReference.compareAndSet(null, lockObj)) {
            return true;
        }
        // 判断锁是否失效, 避免发生死锁, 如果失效再次尝试获取锁.
        LockObj oldLockObj = lockObjReference.get();
        return (oldLockObj == null || oldLockObj.expireTime < System.currentTimeMillis())
                && lockObjReference.compareAndSet(oldLockObj, lockObj);
    }

    /**
     * 释放锁
     * 如果返回值为false, 说明当前用户没有拥有该锁, 或者曾经拥有锁但是因为锁过期而被别的用户获取了
     * @param owner 拥有锁的用户
     * @return 释放锁成功返回true, 否则返回false
     */
    public boolean unLock(String owner) {
        LockObj currentLockObj = lockObjReference.get();
        // 如果锁的拥有者是owner参数指定的用户, 才能解锁
        // 不能直接用set方法, 需要原子更新, 因为有可能锁过期突然被别的用户获取了
        return currentLockObj != null && currentLockObj.owner.equals(owner)
                && lockObjReference.compareAndSet(currentLockObj, null);
    }
}

```

