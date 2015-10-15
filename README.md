> 首先，每个Option对应每一项设置
       
>       读取设置值 Options.read()

>       更改设置值 Options.write()

> 然后将所有设置放到一个Map<String, Options>中管理
 
> 优点：

> 灵活，方便整体操作，如backup、restore

> 扩展性高，可以扩展成类似权限模型配置的工具，如管理员权限模型、普通用户权限模型，识别身份后可直接进行对应权限设置


```
public class Option {
  private final String fieldName;
  private final String setterName;
  private final String getterName;
  private final class clazz;
  private int mDefaultRes = -1;

  private Object readFromField(ClassObject classObject) {
    Field field = classObject.getClass().getDeclaredField(fieldName);
    field.setAccessible(true);
    return field.get(clazz);
  }
     
  private Object readFromGetter(ClassObject classObject) {
    Method method = classObject.getClass().getDeclaredMethod(getterName);
    method.setAccessible(true);
    return method.invoke(clazz);
  }

  private void writeBySetter(ClassObject classObject) {
    Method method = classObject.getClass().getDeclaredMethod(setterName,
                            Context.class, clazz, OnConfigChangeListener listener);
    method.setAccessible(true);
    method.invoke(classObject, context, newValue, listener);
  }
    
  private void writeByField(ClassObject classObject) {
    Field field = classObject.getClass().getDeclaredField(fieldName);
    field.setAccessible(true);
    field.set(this, newValue);
  }

  public Object read(ClassObejct classObject) {
    if(TextUtils.isEmpty(getterName) {
      return readFromField(classObject);
    } else {
      return readFromGetter(classObject);
    }
  }

  public void write(ClassObject classObject) {
    if(TextUtils.isEmpty(setterName) {
      return writeByField(classObject);
    } else {
      return writeBySetter(classObject);
    }
  }

  public void setDefault(int res) {
    this.mDefaultRes = res;
    return this;
  }
    
  public Object getDefault(Resources resources) {
    if(boolean.class.isAssignableFrom(clazz)) {
      return resources.getBoolean(mDefaultRes);
    } else if(int.class.isAssignableFrom(clazz) ){
      return resources.getint(mDefaultRes);
    } else if(String.class.isAssignableFrom(clazz)) {
      return resources.getString(mDefaultRes);
    }
  }
}
```

> config_defaults.xml

```
<resources>
  <bool name="field_boolean">true</bool>
  <integer name="field_int">1</integer>
  <string name="field_string">goodbyeworld</string>
</resources>
```

> Config.java

```
public class Config {
    
  private boolean field_boolean;
  private static final String key_field_boolean = "field_boolean";
  private String field_string;
  private static final String key_field_string = "field_string";
  private int field_int;
  private static final String key_field_int = "field_int";

  Map<String, Option> optionsMap;

  //each field's setters and gettes, like
  public void setFieldBoolean(Context context, boolean fieldBoolean, 
                                      OnConfigChangedListener listener) {
    //todo...
  }

  private void createOptionsMap() {
    optionsMap.put(key_field_boolean, new Option("field_boolean", "setFieldBoolean",                      
                                      "getFieldBoolean", boolean.class))
                                      .setDefault(R.bool.field_boolean);
    optionsMap.put(key_field_string, new Option("field_string", "setFieldString", 
                                      "getFieldString", String.class))
                                      .setDefault(R.string.field_string);
    optionsMap.put(key_field_int, new Option("field_int", "setFieldInt", 
                                      "getFieldInt", int.class))
                                      .setDefault(R.int.field_int);
    }
  }
```

> init

```
  void init() {
    Resources res = context.getResources();
      for(Map.Entry<String, Option> entry : getMap().entrySet()) {
        String key = entry.getKey();
        Option options = entry.getValue();
    }
  }
```

> backup

```
  void backup() {
    JSONObject json = new JSONObject();
    for(Map.Entry<String, Option> entry : getMap().entrySet()) {
      json.put(entry.key(), entry.value());
    }
    //save json to file or, etc.
  }
```

> restore

```
  void restore() {
    JSONObject json = new JSONObject(jsonStr);
    Iterator<String> iter = json.keys();
    while(iter.hasNext()) {
      String key = iter.next();
      Object value = json.get(key);
      Option option = getMap().get(key);
      option.write(classObject); // here classObject is Config
    }
  }
```
