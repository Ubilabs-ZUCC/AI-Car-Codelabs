刚才只是撘了一个框架，使你的小车能和外界（手机）进行通信，但是仅仅是与外界保持通信而已。就像手机说了一句话：“小车你左转”。小车的确能听到，但是却无法根据这个命令执行相应的动作。那接下来我们就来补充刚才那段代码，使小车不仅有能力听懂手机的话而且能按照手机的指示完成相应的动作。

首先在全局变量区域填写以下必要的全局变量

```c
int op=0;
int wheel=2;
long last[5]={0};//定时1
int pulse_listen[2]={0};
int pulse_change[4]={0};
int L_speed[4]={0};
int R_speed[4]={0};
int count_dif[5]={0};//0放总偏差，1放50毫秒偏差
int execute_straight=2;
int command_s=0;
int PWM_temp[2]={0};
int PWM[2]={0};//PWM[0]为L的PWM值，调直函数用到。
int DT=50;//测速间隔（毫秒）
int Dcount[2]={0};
int adjust_count=0;
double PID[2]={0.5,1};
//调试变量

int Rtt=0;
int Ltt=0;
int Dtt=0;
int Stt=0;
```

在接收数据json处理区内填上如下代码

```c
 int L=root["L"];
   int R=root["R"];
   int S=root["S"];
   int D=root["D"];
   command_s=S;
   Ltt=L;
   Rtt=R;
   Stt=S;
   Dtt=D;
   translate(D,S,L,R);
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

```

在loop代码区域填写如下代码，这个是主循环函数，其中包括码盘的测量，电动机的控制函数（这些函数接下去会提到）

```c

       if(op<=4){
        seek_wheel();
        }

        pulse_num();
        
        straight(execute_straight);
```

接下去最后一步，在代码的最下面有一片函数区域，填写必要的函数。第一个是走直算法straight这个算法目前还不是非常完美。

然后是execute函数，用来对电机下达指令控制电机的转动。empty\_values函数是每一个指令完成后变量复原函数。pulse\_num函数用于时刻监听左右码盘的转速。translate函数在接收到手机发送的json指令并解析后做出电机执行的策划,它封装了包括左转，右转，直行，后退等的功能。

```c

void seek_wheel(){
     
     long now_setup = millis();
     if(now_setup-last[1]>500){
      op++;
      last[1]=millis();
     }
     if(op==2){
      execute(0,1,1023,1023);
     }
     if(op==3){
        pulse_num();
     }
     if(op==4){  execute(0,1,0,0);
        if(count_dif[0]>0){
            wheel=0;//定左轮
           }else{
            wheel=1;//定右轮
         }
     }
}

void ept_values(int ept){
  if(ept==1){//全部初始化（除execute_straight,command_s）
    L_speed[0]=0;
    R_speed[0]=0;
    L_speed[1]=0;
    R_speed[1]=0;
    L_speed[2]=0;
    R_speed[2]=0;
    count_dif[0]=0;
    count_dif[1]=0;
    adjust_count=0;
    PWM_temp[1]=0;
    PWM_temp[0]=0;
    PWM[0]=0;
    PWM[1]=0;
    DT=50;
    Dcount[0]=0;
    Dcount[1]=0;
  }
  else if(ept==2){//测速时间变化时使用
     L_speed[0]=0;
    R_speed[0]=0;
    L_speed[1]=0;
    R_speed[1]=0;
    L_speed[2]=0;
    R_speed[2]=0;
    count_dif[0]=0;
    count_dif[1]=0;
    PWM_temp[1]=0;
    PWM_temp[0]=0;
    command_s=0;
  }
  else{
    //do nothing
  }
}

void translate(int D,int S,int L,int R){//处理发过来的指令
   if(S<0) {  S=0-S; }
   
   if(D==0){
   execute_straight=2;
   ept_values(1);
   execute(1,1,0,0);
    DT=50;
   }
   else if(D==1){
    ept_values(1);
    execute_straight=1;
     DT=50;
   }
   else if(D==2){
    ept_values(1);
    execute_straight=0;
     DT=50;
   }
   else if(D==3){
    execute_straight=2;
    execute(0,1,S,S);
     DT=50;
     ept_values(1);
   }
   else if(D==4){
    execute_straight=2;
    execute(1,0,S,S);
     DT=50;
     ept_values(1);
   }
  else if(D==5){
     DT=50;
    execute_straight=2;
    bool execute_l=1;
    bool execute_r=1;
    if(L<0)
    {
    L=-L;
    execute_l=0;
    }
    if(R<0)
    {
      execute_r=0;
      R=-R;
      }
    execute(execute_l,execute_r,L,R);
   }
}

void execute(bool execute_l,bool execute_r,int l_speed,int r_speed)//电机执行函数，execute_l为1，左电机前进，为0则退
{
  digitalWrite(D1,0+execute_r);
  digitalWrite(D0,!(0+execute_r));
  analogWrite(D3,r_speed);
  digitalWrite(D2,0+execute_l);
  digitalWrite(D4,!(0+execute_l));
  analogWrite(D5,l_speed);
}

void straight(int dir){//走直算法
  if(adjust_count>=3){
    if(count_dif[1]<0&&Dcount[0]>Dcount[1]){
          if(wheel==0){//定左轮
            PWM[1]= PWM[1]+abs(count_dif[1])*PID[1];
          }
          else{
            PWM[0]=PWM[0]-abs(count_dif[1])*PID[1];
          }
                    Dcount[1]=Dcount[0]+1;
          }
      else if(count_dif[1]>0&&Dcount[0]>Dcount[1]){
              if(wheel==1){//定右轮
            PWM[0]= PWM[0]+abs(count_dif[1])*PID[1];
              }
              else{
              PWM[1]= PWM[1]-abs(count_dif[1])*PID[1];
              }
                 Dcount[1]=Dcount[0]+1;
              }
            execute(dir,dir,PWM[0],PWM[1]);
  }


else if(adjust_count<3){
  if(dir==2){
    //do nothing
  }
  else if(dir==1||dir==0){
    if(wheel==0){//定左轮
      PWM[0]=command_s;
     if(adjust_count==0){
      PWM[1]=command_s;
      adjust_count++;Dcount[1]=Dcount[0];
     }
      if(count_dif[1]>0&&Dcount[0]>Dcount[1]){//右轮快
   //   PWM[1]=command_s-count_dif[1]*PID[0];
        adjust_count++;Dcount[1]=Dcount[0]+1;
      }
      else if(count_dif[1]<0&&Dcount[0]>Dcount[1]){//右轮慢

      // PWM[1]=command_s-count_dif[1]*PID[0];
        adjust_count++;Dcount[1]=Dcount[0]+1;
      }
    }

    if(wheel==1){//定右轮
      PWM[1]=command_s;
      if(adjust_count==0){
      PWM[0]=PWM[0]+count_dif[1]*PID[0];
      adjust_count++;Dcount[1]=Dcount[0];
     }
      if(count_dif[1]>0&&Dcount[0]>Dcount[1]){//左轮慢
      PWM[0]=PWM[0]+count_dif[1]*PID[0];
        adjust_count++;Dcount[1]=Dcount[0]+1;
      }
      else if(count_dif[1]<0&&Dcount[0]>Dcount[1]){//左轮快
        PWM[0]=PWM[0]+count_dif[1]*PID[0];
        adjust_count++;Dcount[1]=Dcount[0]+1;
      }
    }
         execute(dir,dir,PWM[0],PWM[1]);
  }
  else{
    //do nothing
  }
}
}

void pulse_num(){ //测量脉冲
  pulse_listen[0]=digitalRead(D7);
  if(!pulse_change[0]){
     pulse_change[2]= pulse_listen[0];
    pulse_change[0]=1;
  }
  if( pulse_listen[0]!= pulse_change[2]){
    L_speed[0]+=1;
    L_speed[1]+=1;
     pulse_change[2]= pulse_listen[0];
  }
  
   pulse_listen[1]=digitalRead(D6);
  if(!pulse_change[1]){
     pulse_change[3]= pulse_listen[1];
    pulse_change[1]=1;
  }
  if( pulse_listen[1]!=pulse_change[3]){
    R_speed[0]+=1;
    R_speed[1]+=1;
    pulse_change[3]= pulse_listen[1];
  }
  count_dif[0]=R_speed[0]-L_speed[0];
 detection_count(DT);
}

void detection_count(int Delta_T){   //每 Delta_T毫秒测一次码盘数据，发布版在_speed[2]中
  long speed_detection_time_now=millis();
  long Delta_T_Real=speed_detection_time_now-last[0];
  if(Delta_T_Real>Delta_T){
        R_speed[2]=R_speed[1]*Delta_T/(Delta_T_Real);  
        L_speed[2]=L_speed[1]*Delta_T/(Delta_T_Real);
        R_speed[1]=0; 
        L_speed[1]=0;
        count_dif[1]=R_speed[2]-L_speed[2];
        last[0]=speed_detection_time_now;
        Dcount[0]++;
    }
  }

```

到这一步，小车的代码已经完成，插上USB线，选好端口点上传键![](/assets/23import.png)这时小车部分就完成了

