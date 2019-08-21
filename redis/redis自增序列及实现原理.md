#### 实现原理： ####
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其实就是将一个整数型的key放入到redis中，不过因为redis只支持string等数据类型，不支持整数型，所以是以字符串类型保存在redis中的，用的时候直接将key值取出，再++之后赋值回去。
#### 代码实现：
```
try {
   //当redis中没有这个key的时候，会创建一个数值为0的key
   RedisAtomicLong counter = new RedisAtomicLong("redis中的key", redisTemplate.getConnectionFactory());
   Long increment = counter.incrementAndGet();
   if (increment == 1) {
       //获取当天时间的最后一秒钟
       Long expireTime = DateUtils.getLastTimeOfDay(new Date());
       //设置过去时间，当第二天的时候这个key因为过期了，所以会重新生成一个从0开始的key
       counter.expireAt(new Date(expireTime));
   }
   //将序列格式化为三位数字
   DecimalFormat decimalFormat = new DecimalFormat("000");
   String dateFormat = new SimpleDateFormat("yyyyMMdd").format(new Date());
   return dateFormat + decimalFormat.format(increment);
} catch (Exception e) {
   logger.error("从redis获取自增序列发生异常,返回UUID", e);
   return UUID.randomUUID().toString();
}
```