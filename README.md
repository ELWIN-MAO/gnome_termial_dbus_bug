# gnome_termial_dbus_bug

```
[abc@arh_test ~]$ gnome-terminal 
Error constructing proxy for org.gnome.Terminal:/org/gnome/Terminal/Factory0: Error calling StartServiceByName for org.gnome.Terminal: Timeout was reached


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
```

是否能通过，静态分析或者符号执行的方式找到terminal_factory_proxy_new_for_bus_sync函数返回NULL的详细路径。
