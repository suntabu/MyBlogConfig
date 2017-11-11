

##### Recently I worked on an android keyboard app, encountered the need of gesture typing path drawing effect. Normalizely, by using drawing path of the records of ontouchevent postion would be ok, but we could do it better like below:

[![](https://raw.githubusercontent.com/suntabu/AndroidPathTail/master/android_path_tail.gif)](https://www.youtube.com/watch?v=wB3i_P31Sek)

Let's get over the code:

1. The main logical of drawing path tail is in class [**PathTailHelper.java**](https://github.com/suntabu/AndroidPathTail/blob/master/app/src/main/java/com/suntabu/pathtail/PathTailHelper.java)

2. Then use the instance of PathTailHelper in any view class which would be drawn with the tail. In my project it was [**PathTailView.java**](https://github.com/suntabu/AndroidPathTail/blob/master/app/src/main/java/com/suntabu/pathtail/PathTailView.java)

``` java


```

