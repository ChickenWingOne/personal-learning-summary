### 方法一 ###
```
File sourceFile = ResourceUtils.getFile("classpath:picture/bottom.png"); //这种方法在linux下无法工作
```

### 方法二 ###
```
Resource resource = new ClassPathResource("picture/bottom.png");
File sourceFile =  resource.getFile();
```