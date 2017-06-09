---
title: Fragment---监听返回键事件
date: 2016-08-09
tags: 事件传递管理
categories: 事件监听
---

##### 有时候为了方便代码维护，在fragment中可能要处理一些监听fragment back键
```
//主界面获取焦点
    private void getFocus() {
        getView().setFocusableInTouchMode(true);
        getView().requestFocus();
        getView().setOnKeyListener(new View.OnKeyListener() {
            @Override
            public boolean onKey(View v, int keyCode, KeyEvent event) {
                if (event.getAction() == KeyEvent.ACTION_UP && keyCode == KeyEvent.KEYCODE_BACK) {
                    // 监听到返回按钮点击事件
                    ......
                   return true;
                }
                return false;
            }
        });
    }
```
>以上代码是stackoverflow.com中找到的一个解决方案，但是在使用时，由于Fragment页面里可能有其他能获取焦点的View（例如EditText），会导致监听失效，点击返回键会返回到上个页面。

##### 更完善的解决方案：
除了上面的代码，我们需要对可以获取焦点的View的setOnKeyListener进行处理，这里以一个EditText为例：

```
//private EditText nickname;

nickname.setOnKeyListener(new View.OnKeyListener() {
            @Override
            public boolean onKey(View v, int keyCode, KeyEvent event) {
                if (keyCode == KeyEvent.KEYCODE_BACK
                        && event.getAction() == KeyEvent.ACTION_UP) {
                    //关闭软键盘
                    InputMethodManager imm = (InputMethodManager) getActivity().getSystemService(Context.INPUT_METHOD_SERVICE);
                    imm.hideSoftInputFromWindow(nickname.getWindowToken(), 0);
                    //使得根View重新获取焦点，以监听返回键
                    getFocus();
                }
                return false;
            }
        });
```

##### 使用到的资料：
>[http://blog.csdn.net/ccpat/article/details/45176665](http://blog.csdn.net/ccpat/article/details/45176665)
[http://stackoverflow.com/questions/22552958/handling-back-press-when-using-fragments-in-android](http://stackoverflow.com/questions/22552958/handling-back-press-when-using-fragments-in-android)