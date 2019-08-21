### 引入依赖：
```
<dependency>
    <groupId>com.github.axet</groupId>
    <artifactId>kaptcha</artifactId>
    <version>0.0.9</version>
</dependency>
```
### 创建对应的bean
```
package ecological.environment.intelligence.system.config;

import com.google.code.kaptcha.impl.DefaultKaptcha;
import com.google.code.kaptcha.util.Config;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import java.util.Properties;

/**
 * @author: JW
 * @date: 2019/2/11
 */
@Component
public class KaptchaConfig {
    @Bean
    public DefaultKaptcha captchaProducer() {
        DefaultKaptcha captchaProducer = new DefaultKaptcha();
        Properties properties = new Properties();
        properties.setProperty("kaptcha.border", "no");
        properties.setProperty("kaptcha.border.color", "105,179,90");
        properties.setProperty("kaptcha.textproducer.font.color", "black");
        properties.setProperty("kaptcha.image.width", "160");
        properties.setProperty("kaptcha.image.height", "45");
        properties.setProperty("kaptcha.textproducer.font.size", "35");
        properties.setProperty("kaptcha.session.key", "code");
        properties.setProperty("kaptcha.textproducer.char.length", "6");
        properties.setProperty("kaptcha.textproducer.char.string", "0123456789");
//        properties.setProperty("kaptcha.textproducer.font.names","宋体,楷体,微软雅黑");
//        properties.setProperty("kaptcha.noise.color", "0,255,255");
        Config config = new Config(properties);
        captchaProducer.setConfig(config);
        return captchaProducer;
    }
}
```
### Controller层应用
```
public void getCaptcha(HttpServletRequest request, HttpServletResponse response){
    HttpSession session = request.getSession();
    response.setDateHeader("Expires", 0);
    response.setHeader("Cache-Control", "no-store, no-cache, must-revalidate");
    response.addHeader("Cache-Control", "post-check=0, pre-check=0");
    response.setHeader("Pragma", "no-cache");
    response.setContentType("image/jpeg");
    String text = captchaProducer.createText();
    //Constants是kaptcha自带的
    session.setAttribute(Constants.KAPTCHA_SESSION_KEY, text);
    // create the image with the text
    BufferedImage bi = captchaProducer.createImage(text);
    try {
        ServletOutputStream out = response.getOutputStream();
        ImageIO.write(bi, "jpg", out);
    } catch (Exception e) {
        log.error("获取图片验证码发生异常", e);
    }
}
```