# Android基础常识

1. dp, dip, sp, px, pt，ppi，dpi这些单位的区别？
   - px：pixel，像素，电子屏幕上的基本单元
   - pt：point，等于1/72英寸
   - ppi：pixel per inch，每英寸像素数
   - dpi：dot per inch，每英寸多少点
   - dp：dip，density-independent pixel，在屏幕像素密度为160ppi的时候1dp=1px
   - sp：scale-independent pixel,安卓开发用的字体大小单位。
2. drawable-mdpi, drawable-hdpi, drawable-xhdpi, drawable-xxdpi，drawable-xxxdpi的区别是什么？
   mdpi对应的ppi是160
   hdpi对应的ppi是240
   xhdpi对应的ppi是320
   xxhdpi对应的ppi是480

3. 介绍下Activity的启动模式？

4. Android系统的架构图？

5. 介绍下Android各版本的区别？

6. RecyclerView中的item宽高失效的原因深究？

7. RecyclerView如何通过notifyDataSetChanged通知数据更新的？

8. ViewHolder的作用是什么？它是如何进行优化的？

9. include，merge, ViewStub标签的作用？
    - include标签是为了提高布局的复用，便于相同视图内容的统一控制管理
    - merge标签可以优化布局，消除视图层级中多余的Layout
    - ViewStub就是当你需要的时候才会加载，可以减少内存的使用。
10. leakCannary的原理？

11. art和dalvik的区别？
    - 引入了Ahead-of-time技术，直接在安装的时候把dex翻译成机器码，使得系统性能显著提升，应用启动更快，运行更快，体验更流畅。缺点是要占用更大的存储空间，还需要更长的安装时间。
    - 优化了垃圾回收机制
12. Android的打包编译流程？
    ![Android编译流程](assets/android_compiler.png)
