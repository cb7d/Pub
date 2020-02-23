无痕埋点思路

hook UIViewController 声明周期

构造视图响应链条，从被点击的view到controller，标记类名，

cell等采用index标识

通过md5加密生成唯一id标识

服务端解析埋点文件，匹配运营、产品给出的埋点位置

