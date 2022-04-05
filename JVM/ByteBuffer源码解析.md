# ByteBuffer源码解析

1. ByteBuffer之DirectByteBuffer源码分析

   - ByteBuffer的另一个子类DirectByteBuffer，直接在堆外分配内存。(直接内存)

   - ByteBuffer.allocateDirect（int）创建DirectByteBuffer对象

     ```java
     // allocate 分配 direct直接 就分配直接内存
     public static ByteBuffer allocateDirect(int capacity) {
         return new DirectByteBuffer(capacity);
     }
     ```

   - 查看New DirectByteBuffer(capacity)

     ```java
     DirectByteBuffer(int cap) {                   // package-private(包访问路径)
     
         super(-1, 0, cap, cap);
         boolean pa = VM.isDirectMemoryPageAligned();
         int ps = Bits.pageSize();
         long size = Math.max(1L, (long)cap + (pa ? ps : 0));
         Bits.reserveMemory(size, cap);
     
         long base = 0;
         try {
             base = unsafe.allocateMemory(size);
         } catch (OutOfMemoryError x) {
             Bits.unreserveMemory(size, cap);
             throw x;
         }
         unsafe.setMemory(base, size, (byte) 0);
         if (pa && (base % ps != 0)) {
             // Round up to page boundary
             address = base + ps - (base & (ps - 1));
         } else {
             address = base;
         }
         cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
         att = null;
     
     
     
     }
     ```

