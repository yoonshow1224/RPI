```c
#include <Wire.h> 
#include <LiquidCrystal_I2C.h>
LiquidCrystal_I2C lcd(0x27,16,2);


#define mq A0 //mq 센서
#define fan 6 // 팬 조정하는 릴레이
#define aw 7 // 분사기 조정하는 릴레이


int limit = 33; //mq 감지값에 따라 팬 돌리는 기준

// LED 핀 번호
int red = 51; 
int green = 52;
int blue = 53;


void setup()
{
  lcd.init();
  lcd.backlight();
  Serial.begin(9600);
  pinMode(mq, INPUT);
  pinMode(fan, OUTPUT);
  pinMode(aw, OUTPUT);
  pinMode(red, OUTPUT);
  pinMode(green, OUTPUT);
  pinMode(blue, OUTPUT);
}

//LED 조정
void ledlight(int r, int g, int b) {
  digitalWrite(red, r);
  digitalWrite(green, g);
  digitalWrite(blue, b);
}

int c = 0; //분사기 초 조절

void loop()
{
  c+=1; //거의 1초마다 +1

  int mqvalue = analogRead(mq); //MQ 센서 값 저장해두기

  lcd.print("smell level: ");
  lcd.print(mqvalue); //lcd에 프린트

  if(mqvalue>=10 && c>=10) { //1분마다 mq값 50 넘는지 체크
    c = 0; // 초기화
    Serial.print(mqvalue);
    Serial.println(" a"); //시리얼 통신용
    //분사
    digitalWrite(aw, HIGH);
    delay(1000);
    digitalWrite(aw, LOW);
  }
  else {
    Serial.print(mqvalue); // 시리얼 통신용
    Serial.println(" b");
  }
  //mq 값에 따라 LED 색 조정 (50이상 : 빨간색, 35이상 : 파란색, 35미만 : 초록색)
  if (mqvalue>=50) { ledlight(HIGH, LOW, LOW); }
  else if (mqvalue>=35) { ledlight(LOW, LOW, HIGH); }
  else { ledlight(LOW, HIGH, LOW); } 
  //mq 값이 limit(33)을 넘으면 돌리기 
  if (mqvalue>limit) { digitalWrite(fan, HIGH);}
  else { digitalWrite(fan, LOW); }

  delay(1000); //mq센서 1초마다 감지
  lcd.clear();
}
```
```python
import serial
import time
import csv
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from tkinter import Tk, Label
from datetime import datetime

now = datetime.now()


index = 0 #분사 횟수
#Tk 세팅
root = Tk()
root.title("분사")
root.geometry("300x300+500+250")
root.resizable(False, False)
label = Label(root, text="0번 분사됨 \n 3000번 분사 후 교체 필요", font=("Arial", 15))
label.pack(pady=50)

def updaten(): #함수 작동하면 index에 1 더하고 tk에 표시
    count = open("count.txt", "r")
    index = int(count.readline())
    count.close()
    index+=1
    label.config(text = f'{index}번 분사됨\n{3000-index}번 분사 후 교체 필요')


ser = serial.Serial('COM10', 9600, timeout=2) #시리얼 통신
time.sleep(2) #잠깐 기다렸다 시작
#그래프 그리기 세팅
fig = plt.figure()
ax = plt.axes()
ax.set_ylim(0, 125) #표시하는 최댓값은 125

filename = "bt_data.csv"
    


csv_file = open(filename, "a", newline="")
csv_file.truncate(0)    
writer = csv.writer(csv_file)
writer.writerow(["시간", "데이터"])

x = [0] #초
y1 = [0] #센서값
y2 = [0] #분사 그래프

line1, = ax.plot(x, y1)
line2, = ax.plot(x, y2, linestyle='', marker='o', color='blue') #파란 점 그래프
MAX_LEN = 100 #100초 전 것까지 볼 수 있음

def update(num):
    global x, y1, y2
    ax.set_xlim(0, num+1)
    try:
        va = list(ser.readline().decode().strip().split()) #센서 값 읽기
        timestamp = datetime.now().strftime('%Y.%m.%d - %H:%M:%S')
        writer.writerow([timestamp, va[0], 'X' if va[1]=='b' else '분사'])
        csv_file.flush()
    except Exception as err:
        

        
        print('시리얼 읽기 오류남')
        print('이유 :', err)
        exit(0)
    if va[1]=='a':
        count = open("count.txt", "r")
        xx = int(count.readline())
        count.close()
        count = open("count.txt", "w")
        count.truncate(0)
        count.write(str(xx+1))
        count.close()
        print(f'치익({va[0]})')
        updaten()
        x.append(num)
        y1.append(va[0])
        y2.append(40)
    else:
        print(f'{num}초 : {va[0]}')
        x.append(num)
        y1.append(int(va[0]))
        y2.append(0)
    line1.set_data(x[:num], y1[:num])
    line2.set_data(x[:num], y2[:num])
    ax.relim()
    ax.autoscale_view()
    return line1, line2

ani = animation.FuncAnimation(
    fig, update,
    interval=1000,        # 1000ms = 1초
    blit=False,           # blit=False로 하면 그래프 갱신 잘 됨
    repeat=False,
    cache_frame_data=False  # 메모리 캐시 안 함

)
try:
    plt.grid()
    plt.show(block=False)
    root.mainloop()
except:
    ser.close()
    print('종료됨')
csv_file.close()
```
