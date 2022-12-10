# Markdown美化工具


## 美化网站
[https://markdown.com.cn/editor/](https://markdown.com.cn/editor/)  
[http://md.aclickall.com/](http://md.aclickall.com/)  
[https://editor.mdnice.com/](https://editor.mdnice.com/)  
[https://mdx.bioitee.com/](https://mdx.bioitee.com/)  

## 测试美化
## 我是标题2
### 我是标题3
#### 我是标题4

> 我是引用的别人的 话

`我是单行代码1`

```我是单行代码2```

**我是粗体1**  
*我是斜体1， 斜月三星洞*


## 代码美化
注: 普通句子中换行，输入两个或多个空格，加上回车  
注: Markdown 对代码块的语法是开始和结束行都要添加：
```,其中 ` 为 windows 键盘左上角那个，如下 ：


```java
public class MyActivity extends AppCompatActivity {
@Override  //override the function
    protected void onCreate(@Nullable Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
       try {
            OkhttpManager.getInstance().setTrustrCertificates(getAssets().open("mycer.cer");
            OkHttpClient mOkhttpClient= OkhttpManager.getInstance().build();
        } catch (IOException e) {
            e.printStackTrace();
        }
}
```