# gnome_termial_dbus_bug

出错的gnome-termial  3.26.2 版本的源代码链接。
https://git.gnome.org/browse/gnome-terminal/tree/src/terminal.c?h=3.26.2#n400

https://git.gnome.org/browse/gnome-terminal/snapshot/gnome-terminal-3.26.2.zip

gnome-termial 调用的动态链接库glib2对应的版本(2.54.3)的源代码。
https://gitlab.gnome.org/GNOME/glib/tree/5a650925899c7aae62e9ff79e405e72a6adc981c


```
错误描述和执行方法：
在（最近半年的任意版本）archlinux tty1下面执行 startx  /usr/bin/xterm启动xterm
然后在xterm里面执行 gnome-terminal 发现报错如下：

[abc@arh_test ~]$ gnome-terminal 
Error constructing proxy for org.gnome.Terminal:/org/gnome/Terminal/Factory0: Error calling StartServiceByName for org.gnome.Terminal: Timeout was reached

初步分析知道出错的原因是gnome-termial和dbus-daemon进程通信的时候出错了，没有让dbus-deamon进程
帮助其启动起来 org.gnome.Terminal这个服务对应的gnome-termial这个进程。

pacman -Ql gnome-terminal  |less
gnome-terminal软件包中比较重要的文件
gnome-terminal /usr/bin/gnome-terminal
gnome-terminal /usr/lib/gnome-terminal/gnome-terminal-server
gnome-terminal /usr/lib/systemd/user/gnome-terminal-server.service


$cat /usr/lib/systemd/user/gnome-terminal-server.service
[Unit]
Description=GNOME Terminal Server
[Service]
KillMode=process
Type=dbus
BusName=org.gnome.Terminal
ExecStart=/usr/lib/gnome-terminal/gnome-terminal-server



其中gnome-termial出错函数定位。
1
==========
int main()
{
.......
.......
 factory = terminal_factory_proxy_new_for_bus_sync (G_BUS_TYPE_SESSION,
                                                     G_DBUS_PROXY_FLAGS_DO_NOT_LOAD_PROPERTIES |
                                                     G_DBUS_PROXY_FLAGS_DO_NOT_CONNECT_SIGNALS,
                                                     service_name,
                                                     TERMINAL_FACTORY_OBJECT_PATH,
                                                     NULL /* cancellable */,
                                                     &error);
  if (factory == NULL) {
    //如果 terminal_factory_proxy_new_for_bus_sync 函数返回值为NULL则这个进程就以出错的形式退出了。
    if (!handle_factory_error (error, service_name))
      g_printerr ("Error constructing proxy for %s:%s: %s\n",
                  service_name, TERMINAL_FACTORY_OBJECT_PATH, error->message);

    goto out;
  }
 out:
  return exit_code;
}

2
==========
TerminalFactory *
terminal_factory_proxy_new_for_bus_sync (
    GBusType             bus_type,
    GDBusProxyFlags      flags,
    const gchar         *name,
    const gchar         *object_path,
    GCancellable        *cancellable,
    GError             **error)
{
  GInitable *ret;
  ret = g_initable_new (TERMINAL_TYPE_FACTORY_PROXY, cancellable, error, "g-flags", flags, "g-name", name, "g-bus-type", bus_type, "g-object-path", object_path, "g-interface-name", "org.gnome.Terminal.Factory0", NULL);
  if (ret != NULL)
    return TERMINAL_FACTORY (ret);
  else
    return NULL;
    //terminal_factory_proxy_new_for_bus_sync 返回NULL的原因是 g_initable_new返回NULL
}


3
=========

gpointer
g_initable_new (GType          object_type,
        GCancellable  *cancellable,
        GError       **error,
        const gchar   *first_property_name,
        ...)
{
  GObject *object;
  va_list var_args;

  va_start (var_args, first_property_name);
  object = g_initable_new_valist (object_type,
                  first_property_name, var_args,
                  cancellable, error);
  va_end (var_args);

  return object;
  // g_initable_new_valist 返回NULL
}

4.
=========
GObject*
g_initable_new_valist (GType          object_type,
               const gchar   *first_property_name,
               va_list        var_args,
               GCancellable  *cancellable,
               GError       **error)
{
  GObject *obj;

  g_return_val_if_fail (G_TYPE_IS_INITABLE (object_type), NULL);

  obj = g_object_new_valist (object_type,
                 first_property_name,
                 var_args);

  if (!g_initable_init (G_INITABLE (obj), cancellable, error))
    {
      //如果g_initable_init初始化obj出错，则此函数返回NULL
      g_object_unref (obj);
      return NULL;
    }

  return obj;
}


5.
=========
gboolean
g_initable_init (GInitable     *initable,
         GCancellable  *cancellable,
         GError       **error)
{
  GInitableIface *iface;

  g_return_val_if_fail (G_IS_INITABLE (initable), FALSE);

  iface = G_INITABLE_GET_IFACE (initable);

  return (* iface->init) (initable, cancellable, error);
  //这里返回了NULL导致了整个进程结束了。
}

分析到这里分析不下去了，出现了函数指针。人工不太好分析了。
```


是否能通过，静态分析或者符号执行的方式找到terminal_factory_proxy_new_for_bus_sync函数返回NULL的详细路径。
这样也就是找出了上面程序执行错误的原因了。


```
上面各个函数都在glib2这个软件包的源代码里面，对应下面的动态链接库。

[abc@arh_test ~]$ ldd /usr/bin/gnome-terminal  | grep gio
        libgio-2.0.so.0 => /usr/lib/libgio-2.0.so.0 (0x00007fbdc6604000)

[abc@arh_test ~]$ nm -D /usr/lib/libgio-2.0.so.0 | grep -w  g_initable_new
0000000000061880 T g_initable_new
[abc@arh_test ~]$

[abc@arh_test ~]$ nm -D /usr/lib/libgio-2.0.so.0 | grep -w  g_initable_new_valist
00000000000617d0 T g_initable_new_valist

[abc@arh_test ~]$ nm -D /usr/lib/libgio-2.0.so.0 | grep -w  g_object_new_valist
                 U g_object_new_valist


[abc@arh_test ~]$ nm -D /usr/lib/libgobject-2.0.so  | grep g_object_new_valist 
0000000000017290 T g_object_new_valist



[abc@arh_test ~]$ nm -D /usr/lib/libgio-2.0.so.0 | grep -w  g_initable_init 
0000000000061690 T g_initable_init





[abc@arh_test src]$ ldd /usr/bin/gnome-terminal  | grep glib
        libglib-2.0.so.0 => /usr/lib/libglib-2.0.so.0 (0x00007f407cd0d000)


 

[abc@arh_test ~]$ ldd /usr/bin/gnome-terminal  | grep gob
        libgobject-2.0.so.0 => /usr/lib/libgobject-2.0.so.0 (0x00007f79e25b7000)




glib2 /usr/lib/libgio-2.0.so
glib2 /usr/lib/libgio-2.0.so.0
glib2 /usr/lib/libgio-2.0.so.0.5400.3
glib2 /usr/lib/libglib-2.0.so
glib2 /usr/lib/libglib-2.0.so.0
glib2 /usr/lib/libglib-2.0.so.0.5400.3
glib2 /usr/lib/libgmodule-2.0.so
glib2 /usr/lib/libgmodule-2.0.so.0
glib2 /usr/lib/libgmodule-2.0.so.0.5400.3
glib2 /usr/lib/libgobject-2.0.so
glib2 /usr/lib/libgobject-2.0.so.0
glib2 /usr/lib/libgobject-2.0.so.0.5400.3
glib2 /usr/lib/libgthread-2.0.so
glib2 /usr/lib/libgthread-2.0.so.0
glib2 /usr/lib/libgthread-2.0.so.0.5400.3



[abc@arh_test glib]$ ldd /usr/lib/libgio-2.0.so.0
        linux-vdso.so.1 (0x00007fff7bdd3000)
        libgobject-2.0.so.0 => /usr/lib/libgobject-2.0.so.0 (0x00007f08399dc000)
        libgmodule-2.0.so.0 => /usr/lib/libgmodule-2.0.so.0 (0x00007f08397d8000)
        libglib-2.0.so.0 => /usr/lib/libglib-2.0.so.0 (0x00007f08394c3000)
        libpthread.so.0 => /usr/lib/libpthread.so.0 (0x00007f08392a5000)
        libz.so.1 => /usr/lib/libz.so.1 (0x00007f083908e000)
        libresolv.so.2 => /usr/lib/libresolv.so.2 (0x00007f0838e77000)
        libmount.so.1 => /usr/lib/libmount.so.1 (0x00007f0838c21000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f083886a000)
        libffi.so.6 => /usr/lib/libffi.so.6 (0x00007f0838661000)
        libdl.so.2 => /usr/lib/libdl.so.2 (0x00007f083845d000)
        libpcre.so.1 => /usr/lib/libpcre.so.1 (0x00007f08381ea000)
        /usr/lib64/ld-linux-x86-64.so.2 (0x00007f0839fdd000)
        libblkid.so.1 => /usr/lib/libblkid.so.1 (0x00007f0837f9c000)
        libuuid.so.1 => /usr/lib/libuuid.so.1 (0x00007f0837d95000)
        librt.so.1 => /usr/lib/librt.so.1 (0x00007f0837b8d000)
```
