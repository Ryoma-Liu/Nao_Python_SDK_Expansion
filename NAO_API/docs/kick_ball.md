## Nao机器人踢球实例讲解
<a href="../Examples/Nao_kick_ball/Nao_kick_ball.zip" download="Nao_kick_ball.zip">源代码下载</a>

该实例包含color_threshold.py和kick_ball两个文件,其中color_threshold.py用于找到合适的阈值进行图像的二值化，kick_ball.py文件是踢球的主程序。

### color_threshold.py
在filename变量中填入图像文件所在的位置。

```python
#!/usr/bin/env python

import cv2
import numpy as np

def callback(object):
    pass

def Choose_Color():
    filename = 'Your file location'

    image0 = cv2.imread(filename,1)
    img = cv2.cvtColor(image0, cv2.COLOR_BGR2HSV)

    img = cv2.resize(img,(int(img.shape[1] / 2),int(img.shape[0] / 2)))
    
    cv2.imshow("image",img)
    
    cv2.createTrackbar("H_min","image",50,255,callback)
    cv2.createTrackbar("H_max","image",150,255,callback)
    
    cv2.createTrackbar("S_min","image",0,255,callback)
    cv2.createTrackbar("S_max","image",255,255,callback)
    
    cv2.createTrackbar("V_min","image",0,255,callback)
    cv2.createTrackbar("V_max","image",255,255,callback)
    
    while(True):

        H_min = cv2.getTrackbarPos("H_min","image",)
        S_min = cv2.getTrackbarPos("S_min","image",)
        V_min = cv2.getTrackbarPos("V_min","image",)

        H_max = cv2.getTrackbarPos("H_max","image",)
        S_max = cv2.getTrackbarPos("S_max","image",)
        V_max = cv2.getTrackbarPos("V_max","image",)
        
        lower_hsv = np.array([H_min, S_min, V_min])
        upper_hsv = np.array([H_max, S_max, V_max])
        
        mask = cv2.inRange(img,lower_hsv,upper_hsv)
        
        cv2.imshow("mask", mask)

        if cv2.waitKey(1) & 0XFF == 27:
            break

Choose_Color()

```
运行程序后，出现交互界面，拖动HSV滑动条，使球的二值化效果达到最佳，记录下H,S,V的数值大小。

### kick_ball.py

#### 导入库
```python
from naoqi import ALProxy
from PIL import Image
import vision_definitions
import numpy as np
import motion
import cv2
import math
import time
```

#### 设置机器人IP
```python
# 改为你自己的机器人的IP地址
robotIP="192.168.2.115"
PORT = 9559
```

#### 初始化代理
```python
motionProxy = ALProxy("ALMotion", robotIP, PORT)
posture = ALProxy("ALRobotPosture",robotIP, PORT)
camProxy = ALProxy("ALVideoDevice", robotIP, PORT)
speech = ALProxy("ALTextToSpeech",robotIP, PORT)
```

#### 初始化机器人姿态
```python
def Init():
    posture.goToPosture('Stand',0.5)
    setHeadAngle(0, 0.25)
    motionProxy.setStiffnesses("Head", 0.0)
```

#### 获取图像
```python
def getImage(cameraID):

    if (cameraID == 0):  # Top Camera
        camProxy.setCameraParameter("test", 18, 0)
    elif (cameraID == 1):  # Bottom Camera
        camProxy.setCameraParameter("test", 18, 1)

    resolution = vision_definitions.kVGA  # resolution
    colorSpace = vision_definitions.kRGBColorSpace  # 
    fps = 15

    nameId = camProxy.subscribe("test", resolution, colorSpace, fps)  # 
    naoImage = camProxy.getImageRemote(nameId)

    imageWidth = naoImage[0]
    imageHeight = naoImage[1]

    array = naoImage[6] #  binary array of size height * width * nblayers containing image data.
    im = Image.frombytes("RGB", (imageWidth, imageHeight), array)

    im.save("raw_Image.png", "PNG")  
    camProxy.unsubscribe(nameId)
```

#### 图像二值化
```python
def Binarization(image, pattern):
    # Setting the pattern
    lower = []
    upper = []
    if (pattern == "red"):
        lower = np.array([0, 129, 16]) #H_min,S_min,V_min
        upper = np.array([21, 255, 255]) #H_max,S_max,V_max
    elif (pattern == "yellow"):
        lower = np.array([20, 100, 100]) #H_min,S_min,V_min
        upper = np.array([34, 255, 255]) #H_max,S_max,V_max
    elif (pattern == "blue"):
        lower = np.array([110, 70, 70]) #H_min,S_min,V_min
        upper = np.array([124, 255, 255]) #H_max,S_max,V_max
    # BGR to HSV
    hsv = cv2.cvtColor(image, cv2.COLOR_BGR2HSV)
    # Binarization
    mask = cv2.inRange(hsv, lower, upper)

    # Opened the image
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT, (5, 5)) 
    opened = cv2.morphologyEx(mask, cv2.MORPH_OPEN, kernel)

    cv2.imshow("Binarization", opened)
    return opened
```

#### 遍历图像，寻找是否有球
若有，则返回flag为true，(x,y)是目标点的坐标；否则返回false,(0,0)。
```python
def search(cameraID): 
    getImage(cameraID)
    img = cv2.imread("raw-Image.png")
    bina = Binarization(img, "red") # color
    x, y = calcTheLocate(bina)
    if (x == 0 and y == 0):
        flag = False
    else:
        flag = True
    return flag, x, y
```

#### 寻找图像中球的中心点
```python
def calcTheLocate(img):
    col = np.ones(640)  
    row = np.ones(480)  
    colsum = []  
    rowsum = []  
    x = 0
    xw = 0 # w:west
    xe = 0 # e:est
    y = 0
    yn = 0 #n:north
    ys = 0 #s:south
    for i in range(0, 480):  
        product = np.dot(col, img[i][:])  
        colsum.append(product)
    for i in range(0, 480):  
        if (colsum[i] == max(colsum)):
            y = i
            val = max(colsum) / 255
            yn = int(i - val)
            ys = int(i + val)
            break
    for i in range(0, 640):
        product = np.dot(row, img[:, i])
        rowsum.append(product)
    for i in range(0, 640):
        if (rowsum[i] == max(rowsum)):
            x = i
            val = max(colsum) / 255
            xw = int(val - i)
            xe = int(val + i)
            break
    print("locate  x: ", x, xw, xe, "........ locate y :", y, yn, ys)
    
    cv2.circle(img, (x, y), 5, (55, 255, 155), -1)
    cv2.circle(img, (xw, y), 5, (55, 255, 155), -1)
    cv2.circle(img, (xe, y), 5, (55, 255, 155), -1)
    cv2.circle(img, (x, yn), 5, (55, 255, 155), -1)
    cv2.circle(img, (x, ys), 5, (55, 255, 155), -1)
    cv2.putText(img, "center", (x - 20, y - 20),
    cv2.FONT_HERSHEY_SIMPLEX, 0.75, (55, 255, 155), 2)

    cv2.imshow("two", img)
    cv2.imwrite("binalizeImage.png", img)
    # cv2.waitKey(0)
    return x, y
```

#### 计算球和机器人的实际距离
该实例用的是底端的摄像头，若使用顶端的摄像头，需要重新测试校准D_min和D_max的值。
```python
def getDistance(x, y):

    img_height = 480
    img_width = 640

    H = 477.33 # Height of bottom camera
	
	# Calibrate the two parameters for your own case
    D_min = 50 # mm, minimum distance
    D_max = 800 # mm, maximum distance

    beta = 56.3 * math.pi / 180.0  # HFOV

    alpha = math.atan(D_min / H)
    theta = math.atan(D_max / H) - alpha

    diff_theta = (img_height - y) * theta / img_height
    x_real = H * math.tan(alpha + diff_theta) /1000

    y_unit = (x_real * 1000 + D_min) * math.tan(beta / 2)
    y_real = -2 * y_unit * (x - img_width / 2) / img_width / 1000

    turn = math.atan(y_real / x_real)

	return x_real, y_real, turn
```

#### 机器人踢球动作
本实例中，机器人用右脚踢球。
```python
def kick_ball():
    names = list()
    times = list()
    keys = list()

    names.append("HeadPitch")
    times.append([ 1.16000, 2.68000, 3.20000, 4.24000, 5.12000, 6.12000])
    keys.append([ [ 0.04363, [ 3, -0.38667, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ 0.26180, [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.17453, \
            [ 3, -0.17333, 0.06012], [ 3, 0.34667, -0.12023]], [ -0.27925, [ 3, -0.34667, 0.00000], \
                [ 3, 0.29333, 0.00000]], [ -0.26180, [ 3, -0.29333, -0.00575], [ 3, 0.33333, 0.00653]], \
                    [ -0.24241, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("HeadYaw")
    times.append([ 1.16000, 2.68000, 3.20000, 4.24000, 5.12000, 6.12000])
    keys.append([ [ 0.00464, [ 3, -0.38667, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ -0.00149, [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.00311, \
            [ 3, -0.17333, 0.00000], [ 3, 0.34667, 0.00000]], [ -0.04905, [ 3, -0.34667, 0.00000], \
                [ 3, 0.29333, 0.00000]], [ -0.03371, [ 3, -0.29333, -0.00382], [ 3, 0.33333, 0.00434]], \
                    [ -0.02459, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("LAnklePitch")
    times.append([ 1.04000, 1.76000, 2.56000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ 0.03226, [ 3, -0.34667, 0.00000], [ 3, 0.24000, 0.00000]], \
        [ 0.01745, [ 3, -0.24000, 0.00000], [ 3, 0.26667, 0.00000]], [ 0.01745, [ 3, -0.26667, 0.00000], \
            [ 3, 0.52000, 0.00000]], [ 0.03491, [ 3, -0.52000, 0.00000], [ 3, 0.29333, 0.00000]], \
                [ 0.03491, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ 0.11501, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])

    names.append("LAnkleRoll")
    times.append([ 1.04000, 1.76000, 2.56000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ 0.33161, [ 3, -0.34667, 0.00000], [ 3, 0.24000, 0.00000]], \
        [ 0.36652, [ 3, -0.24000, 0.00000], [ 3, 0.26667, 0.00000]], [ 0.36652, \
            [ 3, -0.26667, 0.00000], [ 3, 0.52000, 0.00000]], [ 0.36652, [ 3, -0.52000, 0.00000], \
                [ 3, 0.29333, 0.00000]], [ 0.34732, [ 3, -0.29333, 0.01920], [ 3, 0.33333, -0.02182]], \
                    [ -0.08433, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("LElbowRoll")
    times.append([ 1.08000, 2.60000, 3.12000, 4.16000, 5.04000, 6.04000])
    keys.append([ [ -0.74096, [ 3, -0.36000, 0.00000], [ 3, 0.50667, 0.00000]],\
        [ -1.03396, [ 3, -0.50667, 0.15621], [ 3, 0.17333, -0.05344]], [ -1.36990, \
            [ 3, -0.17333, 0.00000], [ 3, 0.34667, 0.00000]], [ -1.02015, [ 3, -0.34667, -0.11965], \
                [ 3, 0.29333, 0.10124]], [ -0.70722, [ 3, -0.29333, -0.10030], [ 3, 0.33333, 0.11398]], \
                    [ -0.37732, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("LElbowYaw")
    times.append([ 1.08000, 2.60000, 3.12000, 4.16000, 5.04000, 6.04000])
    keys.append([ [ -1.15353, [ 3, -0.36000, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ -0.95411, [ 3, -0.50667, -0.06096], [ 3, 0.17333, 0.02085]], [ -0.90809, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ -1.23023, [ 3, -0.34667, 0.11716], [ 3, 0.29333, -0.09913]], \
                [ -1.55697, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ -1.14441, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])

    names.append("LHand")
    times.append([ 1.08000, 2.60000, 3.12000, 4.16000, 5.04000, 6.04000])
    keys.append([ [ 0.00317, [ 3, -0.36000, 0.00000], [ 3, 0.50667, 0.00000]], [ 0.00328, \
        [ 3, -0.50667, -0.00003], [ 3, 0.17333, 0.00001]], [ 0.00329, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ 0.00317, [ 3, -0.34667, 0.00000], [ 3, 0.29333, 0.00000]], \
                [ 0.00325, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ 0.00187, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])

    names.append("LHipPitch")
    times.append([ 1.04000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ 0.23159, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], [ 0.10580, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.12217, [ 3, -0.17333, 0.00000], \
            [ 3, 0.09333, 0.00000]], [ 0.08433, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], \
                [ 0.09046, [ 3, -0.25333, -0.00614], [ 3, 0.29333, 0.00710]], [ 0.19171, [ 3, -0.29333, -0.01627], \
                    [ 3, 0.33333, 0.01849]], [ 0.21020, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("LHipRoll")
    times.append([ 1.04000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ -0.34366, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], [ -0.36820, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ -0.36820, [ 3, -0.17333, 0.00000], \
            [ 3, 0.09333, 0.00000]], [ -0.36513, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], \
                [ -0.36667, [ 3, -0.25333, 0.00000], [ 3, 0.29333, 0.00000]], [ -0.36513, [ 3, -0.29333, \
                    -0.00153], [ 3, 0.33333, 0.00174]], [ 0.10129, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("LHipYawPitch")
    times.append([ 1.04000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ -0.18097, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], [ -0.25307, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ -0.06285, [ 3, -0.17333, -0.02279], \
            [ 3, 0.09333, 0.01227]], [ -0.05058, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], \
                [ -0.18711, [ 3, -0.25333, 0.02986], [ 3, 0.29333, -0.03457]], [ -0.24386, [ 3, \
                    -0.29333, 0.02058], [ 3, 0.33333, -0.02339]], [ -0.31903, [ 3, -0.33333, 0.00000], \
                        [ 3, 0.00000, 0.00000]]])

    names.append("LKneePitch")
    times.append([ 1.04000, 1.76000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ -0.08727, [ 3, -0.34667, 0.00000], [ 3, 0.24000, 0.00000]], [ -0.08727, \
        [ 3, -0.24000, 0.00000], [ 3, 0.26667, 0.00000]], [ -0.09235, [ 3, -0.26667, 0.00000], \
            [ 3, 0.17333, 0.00000]], [ -0.07973, [ 3, -0.17333, 0.00000], [ 3, 0.09333, 0.00000]], \
                [ -0.07973, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], [ -0.07819, \
                    [ 3, -0.25333, -0.00047], [ 3, 0.29333, 0.00055]], [ -0.07666, [ 3, -0.29333, 0.00000], \
                        [ 3, 0.33333, 0.00000]], [ -0.09208, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("LShoulderPitch")
    times.append([ 1.08000, 2.60000, 3.12000, 4.16000, 5.04000, 6.04000])
    keys.append([ [ 1.48649, [ 3, -0.36000, 0.00000], [ 3, 0.50667, 0.00000]], [ 1.35917, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 1.41746, [ 3, -0.17333, -0.02659], \
            [ 3, 0.34667, 0.05318]], [ 1.59847, [ 3, -0.34667, -0.03988], [ 3, 0.29333, 0.03375]], \
                [ 1.63835, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ 1.50021, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])

    names.append("LShoulderRoll")
    times.append([ 1.08000, 2.60000, 3.12000, 4.16000, 5.04000, 6.04000])
    keys.append([ [ 0.02305, [ 3, -0.36000, 0.00000], [ 3, 0.50667, 0.00000]], [ 0.01998, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.13197, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ 0.11816, [ 3, -0.34667, 0.01381], [ 3, 0.29333, -0.01168]], \
                [ 0.02305, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ 0.03524, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])

    names.append("LWristYaw")
    times.append([ 1.08000, 2.60000, 3.12000, 4.16000, 5.04000, 6.04000])
    keys.append([ [ 0.24435, [ 3, -0.36000, 0.00000], [ 3, 0.50667, 0.00000]], [ 0.23935, \
        [ 3, -0.50667, 0.00500], [ 3, 0.17333, -0.00171]], [ 0.22094, [ 3, -0.17333, 0.00409], \
            [ 3, 0.34667, -0.00818]], [ 0.20253, [ 3, -0.34667, 0.00554], [ 3, 0.29333, -0.00469]], \
                [ 0.19026, [ 3, -0.29333, 0.01227], [ 3, 0.33333, -0.01394]], [ -0.12736, [ 3, -0.33333, 0.00000],\
                     [ 3, 0.00000, 0.00000]]])

    names.append("RAnklePitch")
    times.append([ 1.04000, 1.32000, 1.76000, 2.24000, 2.56000, 2.84000, 3.08000, 3.36000, 3.68000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ 0.08727, [ 3, -0.34667, 0.00000], [ 3, 0.09333, 0.00000]], [ -0.08727, \
        [ 3, -0.09333, 0.08824], [ 3, 0.14667, -0.13866]], [ -0.59341, [ 3, -0.14667, 0.00000], \
            [ 3, 0.16000, 0.00000]], [ -0.40143, [ 3, -0.16000, -0.14312], [ 3, 0.10667, 0.09541]], \
                [ 0.12217, [ 3, -0.10667, 0.00000], [ 3, 0.09333, 0.00000]], [ -0.05236, [ 3, -0.09333, 0.04386], \
                    [ 3, 0.08000, -0.03759]], [ -0.12217, [ 3, -0.08000, 0.00000], [ 3, 0.09333, 0.00000]], \
                        [ 0.24435, [ 3, -0.09333, 0.00000], [ 3, 0.10667, 0.00000]], [ -0.12217, [ 3, -0.10667, 0.12468], \
                            [ 3, 0.14667, -0.17144]], [ -0.64403, [ 3, -0.14667, 0.00000], [ 3, 0.29333, 0.00000]], \
                                [ -0.21991, [ 3, -0.29333, -0.11820], [ 3, 0.33333, 0.13432]], [ 0.11356, [ 3, -0.33333, 0.00000], \
                                    [ 3, 0.00000, 0.00000]]])

    names.append("RAnkleRoll")
    times.append([ 1.04000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ 0.40143, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ 0.10887, [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.13802, \
            [ 3, -0.17333, 0.00000], [ 3, 0.09333, 0.00000]], [ 0.00000, [ 3, -0.09333, 0.00000], \
                [ 3, 0.25333, 0.00000]], [ 0.18097, [ 3, -0.25333, -0.05338], [ 3, 0.29333, 0.06181]], \
                    [ 0.34558, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ 0.05066, \
                        [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RElbowRoll")
    times.append([ 1.00000, 2.52000, 3.04000, 4.08000, 4.96000, 5.96000])
    keys.append([ [ 0.64117, [ 3, -0.33333, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ 1.15353, [ 3, -0.50667, -0.18364], [ 3, 0.17333, 0.06282]], [ 1.38056, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ 1.36062, [ 3, -0.34667, 0.01994], [ 3, 0.29333, -0.01687]], \
                [ 0.96024, [ 3, -0.29333, 0.14120], [ 3, 0.33333, -0.16046]], [ 0.45564, \
                    [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])
    names.append("RElbowYaw")
    times.append([ 1.00000, 2.52000, 3.04000, 4.08000, 4.96000, 5.96000])
    keys.append([ [ 0.99714, [ 3, -0.33333, 0.00000], [ 3, 0.50667, 0.00000]],\
        [ 0.86368, [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.90970, \
            [ 3, -0.17333, 0.00000], [ 3, 0.34667, 0.00000]], [ 0.63205, [ 3, -0.34667, 0.00000], \
                [ 3, 0.29333, 0.00000]], [ 0.84834, [ 3, -0.29333, -0.13498], [ 3, 0.33333, 0.15339]], \
                    [ 1.49714, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RHand")
    times.append([ 1.00000, 2.52000, 3.04000, 4.08000, 4.96000, 5.96000])
    keys.append([ [ 0.00129, [ 3, -0.33333, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ 0.00136, [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 0.00132, \
            [ 3, -0.17333, 0.00001], [ 3, 0.34667, -0.00002]], [ 0.00128, [ 3, -0.34667, 0.00000], \
                [ 3, 0.29333, 0.00000]], [ 0.00133, [ 3, -0.29333, -0.00005], [ 3, 0.33333, 0.00006]], \
                    [ 0.00391, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RHipPitch")
    times.append([ 1.04000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ 0.16265, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], [ -0.39726, \
        [ 3, -0.50667, 0.31826], [ 3, 0.17333, -0.10888]], [ -1.11876, [ 3, -0.17333, 0.00190], \
            [ 3, 0.09333, -0.00102]], [ -1.11978, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], \
                [ -0.78540, [ 3, -0.25333, -0.12796], [ 3, 0.29333, 0.14816]], [ -0.29142, [ 3, -0.29333, -0.15581], \
                    [ 3, 0.33333, 0.17705]], [ 0.21318, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RHipRoll")
    times.append([ 1.04000, 2.56000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ -0.47124, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], [ -0.54001, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ -0.32218, [ 3, -0.17333, -0.09040], \
            [ 3, 0.09333, 0.04868]], [ -0.12276, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], \
                [ -0.36360, [ 3, -0.25333, 0.04547], [ 3, 0.29333, -0.05265]], [ -0.41713, [ 3, -0.29333, 0.00000], \
                    [ 3, 0.33333, 0.00000]], [ -0.05825, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RKneePitch")
    times.append([ 1.04000, 2.56000, 2.84000, 3.08000, 3.36000, 4.12000, 5.00000, 6.00000])
    keys.append([ [ -0.08901, [ 3, -0.34667, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ 1.97575, [ 3, -0.50667, 0.00000], [ 3, 0.09333, 0.00000]], [ 1.97222, [ 3, -0.09333, 0.00353], \
            [ 3, 0.08000, -0.00302]], [ 1.23918, [ 3, -0.08000, 0.26583], [ 3, 0.09333, -0.31013]], \
                [ 0.24435, [ 3, -0.09333, 0.00000], [ 3, 0.25333, 0.00000]], [ 1.53589, [ 3, -0.25333, 0.00000], \
                    [ 3, 0.29333, 0.00000]], [ 0.62430, [ 3, -0.29333, 0.25160], [ 3, 0.33333, -0.28591]], \
                        [ -0.07666, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RShoulderPitch")
    times.append([ 1.00000, 2.52000, 3.04000, 4.08000, 4.96000, 5.96000])
    keys.append([ [ 1.52782, [ 3, -0.33333, 0.00000], [ 3, 0.50667, 0.00000]], [ 1.46033, \
        [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ 1.47413, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ 1.24096, [ 3, -0.34667, 0.00000], [ 3, 0.29333, 0.00000]], \
                [ 1.51862, [ 3, -0.29333, -0.02707], [ 3, 0.33333, 0.03076]], [ 1.54938, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])

    names.append("RShoulderRoll")
    times.append([ 1.00000, 2.52000, 3.04000, 4.08000, 4.96000, 5.96000])
    keys.append([ [ -0.12268, [ 3, -0.33333, 0.00000], [ 3, 0.50667, 0.00000]], \
        [ -0.04138, [ 3, -0.50667, 0.00000], [ 3, 0.17333, 0.00000]], [ -0.14569, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ -0.13955, [ 3, -0.34667, 0.00000], [ 3, 0.29333, 0.00000]], [ -0.14722, \
                [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ -0.03993, [ 3, -0.33333, 0.00000], [ 3, 0.00000, 0.00000]]])

    names.append("RWristYaw")
    times.append([ 1.00000, 2.52000, 3.04000, 4.08000, 4.96000, 5.96000])
    keys.append([ [ -0.08727, [ 3, -0.33333, 0.00000], [ 3, 0.50667, 0.00000]], [ -0.07359, \
        [ 3, -0.50667, -0.00911], [ 3, 0.17333, 0.00312]], [ -0.05058, [ 3, -0.17333, 0.00000], \
            [ 3, 0.34667, 0.00000]], [ -0.06285, [ 3, -0.34667, 0.00000], [ 3, 0.29333, 0.00000]], \
                [ 0.05680, [ 3, -0.29333, 0.00000], [ 3, 0.33333, 0.00000]], [ 0.00149, [ 3, -0.33333, 0.00000], \
                    [ 3, 0.00000, 0.00000]]])
    
    motionProxy.angleInterpolationBezier(names, times, keys)
```

#### 校准机器人的位置
```python
def getReady():
    flag, x, y = search(1) #bottom camera
    if x < 330:
        x, y, turn = getDistanceBottom(x, y)
        print("adjust x:", x, "y:", y)
        speech.say("I am adjusting")
        moveConfig = [["MaxStepFrequency", 1.0]]
        motionProxy.moveTo(0, 1.2 * y, turn, moveConfig)
```

#### 主程序
```python
def main():
    Init()   
    flag = None

    while (flag == None):
        flag, x, y = search(1)
        while (flag == False):
            speech.say("I don't see the ball.")
            motionProxy.moveTo(-0.1, 0.1, math.pi / 3)
            flag, x, y = search(1)
        speech.say("I see the ball")

    print("final locate : ", x, y)

    x, y, turn = getDistance(x, y)

    print("walk 0:", x, y, turn)
    moveConfig = [["MaxStepFrequency", 1.0]]
    motionProxy.moveTo(0.68 * x, 0.85 * y, 0.8 * turn, moveConfig)

    getReady()
    speech.say("I am ready")
    kick_ball()

    posture.goToPosture('Stand',0.5)
    
    if __name__ == '__main__':
	    main()
```