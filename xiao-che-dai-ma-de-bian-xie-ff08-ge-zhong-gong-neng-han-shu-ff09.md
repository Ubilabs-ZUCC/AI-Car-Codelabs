刚才只是撘了一个框架，使你的小车能和外界（手机）进行通信，但是仅仅是与外界保持通信而已。就像手机说了一句话：“小车你左转”。小车的确能听到，但是却无法根据这个命令执行相应的动作。那接下来我们就来补充刚才那段代码，使小车不仅有能力听懂手机的话而且能按照手机的指示完成相应的动作。

首先在全局变量区域填写以下必要的全局变量

```
 bool right=1;
bool left=1;
bool s_1=1;
long last_pulse_listen=0;
long error_time_now = 0;
int Direction,RPWM,LPWM,ept,ept_all,al,ar,ir,jr,il,wheel,jl,rn,ln,fl,fr=0;
int L_speed[2];
int R_speed[2];
int dir=100;
int change_count,change_count_2=0;
int loss[10];
long speed_detection_time_last=0;
```

在接收数据json处理区内填上如下代码

```c
    int lpwm=root["L"];
    int rpwm=root["R"];
   int Speed=root["S"];
   Direction=root["D"];
   LPWM= lpwm;RPWM=rpwm;
   translate(Direction,Speed);
```

这段代码的功能是解析出手机发过来的JSON数据格式的指令，将其包含的信息放入对应的变量中去。

找到setup代码区域，将一些必要的初始化代码填入其中

```c
   Serial.begin(9600);
  pinMode(D0,OUTPUT); 
  pinMode(D1,OUTPUT);
  pinMode(D2,OUTPUT);
  pinMode(D3,OUTPUT);
  pinMode(D4,OUTPUT);
  pinMode(D6,INPUT);
  pinMode(D5,OUTPUT);
  pinMode(D7,INPUT);
for(int k=0;k<10;k++){
  loss[k]=0;
}
loss[8]=100000;
```

在loop代码区域填写如下代码，这个是主循环函数，其中包括码盘的测量，电动机的控制函数（这些函数接下去会提到）

```c
   if(Direction!=1&&Direction!=2){
  execute(left,right);
  pulse_num();
  empty_values();
  }
  else{
  straight(!(Direction-1));
  }
```

接下去最后一步，在代码的最下面有一片函数区域，填写必要的函数。第一个是走直算法straight这个算法目前还不是非常完美。

然后是execute函数，用来对电机下达指令控制电机的转动。empty\_values函数是每一个指令完成后变量复原函数。pulse\_num函数用于时刻监听左右码盘的转速。translate函数在接收到手机发送的json指令并解析后做出电机执行的策划,它封装了包括左转，右转，直行，后退等的功能。

```c
  void straight(bool back_forw){
  if(back_forw==0||back_forw==1&&dir==0){
  long now_pulse_listen = millis();
  long error_time_now = millis();
  int loss_step=1;
  int test_time_1=50;//定轮时间参数
  int test_time_2=5;//调步时间参数
  if(0<=al&&al<=200){
    al=2;ar=2;
  }
  empty_values();
  execute(back_forw,back_forw);
  pulse_num();
  loss[0]=ln-rn;
  loss[3]=L_speed[1]-R_speed[1];
if(s_1){  
                      if (now_pulse_listen - last_pulse_listen > test_time_1) {//第一个参数
                          if(change_count_2==0){
                              last_pulse_listen = now_pulse_listen;change_count_2=1;
                          }
                          else if(change_count_2==1){
                            loss[2]=loss[0];
                            if(loss[2]>0&&change_count_2==1){//左边快，左边调小
                              wheel=-1;s_1=0;change_count=0;change_count_2=0;
                            }else if(loss[2]<0&&change_count_2==1){//右边快，右边调小
                              wheel=1;s_1=0;change_count=0;change_count_2=0;
                            }
                            last_pulse_listen = now_pulse_listen;
                          }
                        }
}else{
                      if (now_pulse_listen - last_pulse_listen > test_time_2) {//第二个参数
                            ept=1;
                            loss[1]=loss[0];loss[5]=loss[5]+loss[1];
                            if(loss[1]>0&&wheel==-1&&loss[5]>=0){//左边快，左边调小
                             al=al-loss_step;//第三个参数
                             change_count++;loss[5]=0;
                            }else if(loss[1]<0&&wheel==1&&loss[5]<=0){//右边快，右边调小
                             ar=ar-loss_step;//第4个参数
                             change_count++;loss[5]=0;
                            }
                            else if(change_count<=1){
                              s_1=1;
                              loss[2]=loss[0];
                              if(loss[2]>0){//左边快，左边调小
                                wheel=-1;s_1=0;change_count=0;loss[5]=0;
                              }else if(loss[2]<0){//右边快，右边调小
                                wheel=1;s_1=0;change_count=0;loss[5]=0;
                              }
                            }
                            last_pulse_listen = now_pulse_listen;
                        }
                }
       }
}

void execute(bool execute_l,bool execute_r ){
  digitalWrite(D1,0+execute_l);
  digitalWrite(D0,!(0+execute_l));
  analogWrite(D3,ar);
  digitalWrite(D2,0+execute_r);
  digitalWrite(D4,!(0+execute_r));
  analogWrite(D5,al);
}
void empty_values(){
  if(ept){
    rn=0;ln=0;
    ept=0; 
  }
if(ept_all){
     rn=0;ln=0;
     for(int k=0;k<10;k++){
      loss[k]=0;
       }
    loss[8]=100000;
    ept_all=0;
  }
}

void pulse_num(){
  il=digitalRead(D7);
  if(!fl){
    jl=il;
    fl=1;
  }
  if(il!=jl){
    ln+=1;
    L_speed[0]+=1;
    jl=il;
  }
  ir=digitalRead(D6);
  if(!fr){
    jr=ir;
    fr=1;
  }
  if(ir!=jr){
    rn+=1;
    R_speed[0]+=1;
    jr=ir;
  }
  int Delta_T=50;
  long speed_detection_time_now=millis();
  long Delta_T_Real=speed_detection_time_now-speed_detection_time_last;
  if(Delta_T_Real>Delta_T){
        R_speed[1]=R_speed[0]*Delta_T/(Delta_T_Real);  
        L_speed[1]=L_speed[0]*Delta_T/(Delta_T_Real);
        R_speed[0]=0; 
        L_speed[0]=0;
        speed_detection_time_last=speed_detection_time_now;
}
}

void translate(int Direction,int Speed){
   if(Direction==0){
    al=0;ar=0;right=1;left=1;
   }
   else if(Direction==1||Direction==2){
   al=Speed;
   ar=Speed;
   dir=0;s_1=1;change_count_2=0;change_count=0;wheel=0;ept_all=1;
   }
   else if(Direction==3){
     al=Speed;left=1;
     ar=Speed;right=0;
     ept_all=1;
     dir=100;change_count_2=0;change_count=0;wheel=0;
   }
    else if(Direction==4){
     al=Speed;left=0;
     ar=Speed;right=1;
     ept_all=1;
     dir=100;change_count_2=0;change_count=0;wheel=0;
   }
  else if(Direction==5){
     left=1;right=1;
      if(LPWM<0){
        right=0;LPWM=-LPWM;
      }
      if(RPWM<0){
        left=0;RPWM=-RPWM;
      }
     al=LPWM;
     ar=RPWM;
     ept_all=1;
     dir=100;change_count_2=0;change_count=0;wheel=0;
   }
}
```

到这一步，小车的代码已经完成，插上USB线，选好端口点上传键![](/assets/23import.png)这时小车部分就完成了

