# 我的第一个Android程序（贰）

标签（空格分隔）： android

---

话休繁说，承接上文。上文完成了在服务器端的WebService的部署，这一篇就来说说如何使用前文中提到的ksoap2来在Android Apps当中调用前文中提供的WebService服务。

## Ksoap2

首先通过网络获取ksoap的jar包文件，抑或是直接下载ksoap2的java源码，使用Maven工具自己生jar包，都是可行的方式，这里我为了省些力气（当然了，更重要的是在Windows下做这些可以自动编译的工作实在是痛苦至极，所以我就偷懒使用了别人做好的jar包，但如果你使用的linux系统，推荐还是用Maven工具自己生成一个jar包，因为即便是出了问题，你也可以知道在什么地方去解决，所谓知己知彼也）。然后把jar作为第三方工具加入到我们的项目中（具体方法请自行百度）。待上述工作完成之后，就可以进行代码的编写工作了。

## Coding

洒家使用的开发环境是Android Studio，毕竟这才是Google的亲儿子，Eclipse and ADT再怎么说都只是庶出，如果Google驾崩，那也是Android Studio即位（Eclipse and ADT政变？别闹了，洗洗睡吧）。像所有新建项目一样，我们建立一个实验用的项目，名字随便起。然后按照前述方法加入ksoap2的jar包。
在这里需要先明确几个概念。首先是：Android程序的的注入口点是什么？即类似于C语言中的`Main`函数的代码时什么？这个问题的答案应该是：Android的UI线程，其实说的相对通俗、简单，但是没那么准确的话，答案就是Android Activity中的`onCreate`方法。当一个Android Apps启动时，它就从`onCreate`方法开始进入，然后执行相关的代码。第二：在Android程序能够更新操作UI控件的只有UI线程，而在其他的线程当中都无法操作（当然，也不能说的这么绝对，用一些方法也是可以操作UI线程的），这么做的目的是为了确保Android Apps的响应速度。接下来是代码的编写：

``` Java
public class LoginActivity extends ActionBarActivity {

    String IP = "192.168.0.100";

    EditText name;
    EditText pwd;
    Button login;
    TextView tv;

    String retValue;
    int stCode;
    String hash;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        login = (Button)findViewById(R.id.login);
        tv = (TextView)findViewById(R.id.res);
        name = (EditText)findViewById(R.id.name);
        pwd = (EditText)findViewById(R.id.passwd);

        login.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                UserAsyncTask uat = new UserAsyncTask();
                uat.execute(IP);

                tv.setText(hash);
            }
        });

    }


    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.menu_index, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle action bar item clicks here. The action bar will
        // automatically handle clicks on the Home/Up button, so long
        // as you specify a parent activity in AndroidManifest.xml.
        int id = item.getItemId();

        //noinspection SimplifiableIfStatement
        if (id == R.id.action_settings) {
            return true;
        }

        return super.onOptionsItemSelected(item);
    }

    public class UserAsyncTask extends AsyncTask<String, Void, String>{

        @Override
        protected void onPostExecute(String s) {
            super.onPostExecute(s);
            Log.d("Return Value => ", retValue);

            if(retValue.length() == 0){
                Toast.makeText(getApplicationContext(), "请重试", Toast.LENGTH_SHORT).show();
            }
            else {
                String[] para = retValue.split("\\.");
                Log.d("Length of para" ,para.length + "");
                stCode = Integer.parseInt(para[0]);
                if (stCode == 0 && name.getText().toString() != "" && pwd.getText().toString() != "") {
                    hash = para[1];
                    Intent intent = new Intent(LoginActivity.this, IndexScreen.class);
                    intent.putExtra("hash", hash);
                    intent.putExtra("userCode", name.getText().toString());
                    startActivity(intent);
                    finish();
                } else if (stCode == -1013) {
                    ErrorFragment ef = ErrorFragment.newInstance(R.string.error_1013);
                    ef.show(getSupportFragmentManager(), "Invalid Login");
                } else {
                    ErrorFragment ef = ErrorFragment.newInstance(R.string.error_0);
                    ef.show(getSupportFragmentManager(), "Invalid Login");
                }
            }
        }

        @Override
        protected String doInBackground(String... params) {
            String user_name = name.getText().toString();
            String user_pwd = pwd.getText().toString();

            String nameSpace = "http://" + params[0];
            String url = "http://" + params[0] + "/server.php";
            String methodName = "login";
            String soapAction = "http://" + params[0] + "/server.php/login";

            SoapObject request = new SoapObject(nameSpace, methodName);
            request.addProperty("userCode", user_name);
            try {
                request.addProperty("passwd", MD5(user_pwd));
            } catch (Exception e) {
                e.printStackTrace();
            }
            request.addProperty("ip", "192.168.0.54");

            SoapSerializationEnvelope envelope = new SoapSerializationEnvelope(SoapEnvelope.VER10);
            envelope.setOutputSoapObject(request);

            HttpTransportSE se = new HttpTransportSE(url);
            try{
                se.call(soapAction, envelope);
            }
            catch (HttpResponseException e){
                e.printStackTrace();
            }
            catch (IOException e){
                e.printStackTrace();
            }
            catch (XmlPullParserException e){
                e.printStackTrace();
            }

            SoapObject ret = (SoapObject) envelope.bodyIn;
            retValue = ret.getProperty("return").toString();

            return null;
        }

    }

    public static String MD5(String input) throws Exception {
        MessageDigest md5 = null;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (Exception e) {
            System.out.println(e.toString());
            e.printStackTrace();
            return "";
        }

        byte[] byteArray = input.getBytes("UTF-8");
        byte[] md5Bytes = md5.digest(byteArray);
        StringBuffer hexValue = new StringBuffer();
        for (int i = 0; i < md5Bytes.length; i++) {
            int val = ((int) md5Bytes[i]) & 0xff;
            if (val < 16) {
                hexValue.append("0");
            }
            hexValue.append(Integer.toHexString(val));
        }
        return hexValue.toString();
    }

}
```
由上述代码可以看到我定义了两个文本输入控件（EditText），一个按钮（Button），以及一个文本显示控件（TextView，用于显示WebService返回信息的展示）。

代码中比较重要的是`UserAsyncTask`类，因为对于WebService的调用是通过这个类来实现的。该类通过继承`AsyncTask<T, T, T>`来实现异步的网络访问操作，同时AsyncTask是Android 提供的一种很重要的异步线程调用方法，具体的使用方法及详述会在另一篇文章中进行说明。本文集中说明与WebService相关的内容。这主要设计到代码的第101~128行。而96~99行是定义会在WebService当中用的变量，第101行定义了一个SoapObject对象，用于表示一个WebService请求，其中指明了用于处理这个WebService请求的目的网址及相应的处理方法名。第102~108行是根据前述的方法名，为参数赋值。还记得前一篇文章中我们写的soapHandle.class.php文件中的login方法吗？该方法有三个参数，分别是*usercode， passwd以及ip*，所以102~108行的代码就是向这三个参数传参，但不同的是，这里的传参过程会使用SOAP协议罢了。代码的第110~113行就是发起WebService请求了。第127~128行是用于处理WebService返回数据的，因为本身我们所调用的login函数的返回很简单（只是一个状态码和哈希值），所以这里我所采用的处理方式也很简单的，但是如果返回的是数据集的话，就不能使用这种方法了。而是使用如下的这种方式：
``` Java
SoapObject result = (SoapObject) envelope.bodyIn;

//temp为所有记录的集合，其中每一项都是一条记录
Vector<SoapObject> temp = (Vector<SoapObject>)result.getProperty("return");
```
最开始在处理WebService的返回值时，洒家走了很多的弯路，我以为返回值的类型时Json或者是XML的数据格式，但其实都不是，其返回的数据格式很特殊，类似于这样的：
```
methodName={return=[]}
```
如果return的值不为空，中括号内还有更加复杂的内容，我所采取的处理方式利用字符串替换来提取相关的信息，很原始，所以在以后代码的迭代中需要尝试使用新的方法。对了还有一个问题是在做`result.getProperty`时出现的，那时不知道具体的返回值类型，但是也没用`Vector<SoapObject>`，程序总是报错序列化（Serialization）的错误。彼时无奈，只能一步步的调试、差错，所幸皇天不负有心人，最终得到了`Vector<SoapObject>`这样的一种解决方式。

## 小结
程序的编写总是不尽如人意的，出现的错误对于第一次接触的人来说往往是致命性的打击，也许心脏小些的初学者就倒在了这些Bug面前，但是如果你的心脏够大，你会发现：这些错误才是是你快速提升的捷径， 从某种意义上来说，Bug和错误才是习得一种技能的秘籍。天下本没有捷径的，只是走的人多了，曲径才变成了通途。
也许第一次遇到错误，你会不知所措，但是多遇几次你就知道该怎么办了，所谓：久病成医。虽然不太贴切，但是道理是相通的。“形而上者谓之道，形而下者谓之器”。
至此，WebService的服务器端和客户端的实现方式都已经完成了，接下来，我们就来说说AsyncTask。




