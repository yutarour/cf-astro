---
publishDate: 2023-08-12T00:00:00Z
author: Yutaro U.
title: My experience working with a ROV project in High School
excerpt: So many things to do within such a short time with limited resources.
image: ~/assets/images/ROV/rov_teather_extended.JPEG
category: Engineering Projects
tags:
  - projects
  - engineering
  - pcb-design
  - team-projects
metadata:
  canonical: https://yutaro.dev/rov-project
---

## Table of contents:

- [Background](#background)
- [Members, Mentors](#members-mentors)
- [What I Did](#what-i-did)
- [How it Went](#how-it-went)
- [What I Learned/What I Could Have Done Better](#what-i-learnedwhat-i-could-have-done-better)

## Background

![Power supply and DC motor controller on work bench](~/assets/images/ROV/tinkering.jpg)
I have been exposed to tinkering since a sufficiently young age. I have been disassembling computers since I was maybe 7, and I got my first Arduino kit when I was 9. I didn’t do any programming at that time and only did the wiring, but eventually I learned to write code. This paid off, and in my junior year of high school, I got invited to participate in the school’s ROV (remote operated underwater vehicle) team through recommendations by a few of my teachers. Of course, I wouldn’t just let an opportunity that gave me an excuse to tinker with electronics go like that. However, I could have never expected the amount of head bashing and last-minute hardware changes that our team would have to go through just for a component out of our control to fail at competition. There was a lot I learned through working on this ROV.
<br><br>
To learn more about how I got into electronics and more tips, see [this]() article

## Members, Mentors

Before the covid-19 pandemic, the high school I went to had a ROV team. This team was led by Mr. S, a retired engineer who happened to be friends with Mr. L, our now retired engineering teacher. Mr. S wanted to do the ROV project again, so he asked the teachers at our school to recommend some students that showed potential in engineering for the team. I so happened to be one of the lucky students that were picked for this project (most likely because I had been helping write the Arduino and python curriculum for our school). These recommendations netted us a team of about 12 students; consisting of 2 Seniors, 6 Sophomores, and 4 Freshmen, led by a student called A who was a friend of Mr. S. A picked me as the electrical subsystem lead as we had worked together on a bottle rocket project the year before for an engineering class.
<br><br>
Mr. S also brought 3 of his other retired(?) engineering friends with him, consisting of a Geologist/Project Manager, Electrical engineer, and a Machinist. Mr. S and his friends will become quite important later, so keep them in mind. (Side note: Mr.S’s wife would bake brownies for us every week for the meeting. They were very delicious) Now, one would usually expect mentors with a lot of experience to be a good thing. In our case the mentors were both a blessing and a curse at the same time. On one hand, they would help us out when it came to thinking of ways of doing something (as they were seasoned engineers) and help us out with their connections. On the other hand, they wanted us to do things their way. They (especially Mr. S) had this weird engineer pride that they just couldn’t throw away. For example, when I suggested the use of brushless motors for the project, they rejected the idea in favor of the random DC bilge pump motors we had two years ago. They rejected the idea of having a technical interview for the electrical subsystem, which was a simple interview to see if candidates knew basic engineering principles, when picking additional people for the team.
<br><br>
We also had members who had just joined the team to be able to write it on their resume and didn’t do anything (to be fair these people exist in any school engineering team…). Additionally, since this was a high school team, and for most people the highest level of electronics they have worked with are Arduino Unos and breadboards, they did not know certain key protocols and components. It was partially my fault for just going ahead and doing things when they didn’t get done, but I think I could have done a little more work teaching other members in my team about electrical and embedded system design principles.
<br><br>
Although I have been ranting on about negative things all this time, it wasn’t all negative. We had a very fun time going through the engineering design process and discussing ideas with other team members. We also had a lot of fun testing, fixing, and redesigning the actual ROV to make sure that it worked at competition.

## What I did

As the electrical subsystem lead, I mostly worked on integrating all the subsystems together. Here’s a list of what I mostly worked on:

- Choosing which wires would go in the tether (Cat 5 ethernet, 9awg wire x2 (per mentor request))
- Planning how to control all 6 of the DC motors.
- Creating a custom PCB to power all the motor drivers
- Writing matrix transformation code to decide how much to turn each motor on by
- Configuring DHCP on the raspberry pi to be able to have the pi on a static IP while connected to a unmanaged network switch
- Making the Raspberry pi talk to the PCA9685 PWM multiplexer and writing a wrapper for the Adafruit python-based drivers to talk to all the motors at once.
- Write a API frontend in flask to interface with the Flask backend (that lived on the pi inside of the ROV)
- Write unit tests for the matrix transformation program
- Convert Xbox controller input to motor power
- Help with camera integration
- And some other small things
  <br><br>

### PCA python interface code:

![2 channel motor controller boards hooked up to the pi](~/assets/images/ROV/basic_wiring.jpg)
<br>
Above: Image of the brushed DC motor contoller boards hooked up to the Pi. Wiring: Pi -> PCA9685 -> DC motor driver
<br>

`PCA_interface.py`

```py
"""
Written By Yutaro U.
The piece of code that interfaces the pi to the lower level PCA
pip3 install adafruit-circuitpython-pca9685
https://docs.circuitpython.org/projects/pca9685/en/latest/
"""

from board import SCL, SDA
import busio
from adafruit_pca9685 import PCA9685
import logging
import time

class pca_backend:
    def __init__(self) -> None:
        self.numMotors = 7
        self.i2c_bus = busio.I2C(SCL, SDA)
        self.pca = PCA9685(self.i2c_bus)
        self.pca.reset()
        self.pca.frequency = 60
        self.logger = logging.getLogger()
        self.logger.info("PCA Backend class initialized")

    def setDutyCycle(self,index:int,duty_cycle:int):
        """takes the index from 0 to 15 and sets the duty cycle 0-65535
        16 bit integer is the size limit for this. Only will use 12 bits of resolution
        """
        self.pca.channels[index].duty_cycle=duty_cycle
        self.logger.debug(f"Set duty cycle on {index} to {duty_cycle}")

    def rampDutyCycle(self,index:int,start:int,end:int,seconds:float=0.1):
        """takes the index from 0 to 15 and ramps the duty cycle from start to end in second amount of seconds
        16 bit integer is the size limit for this. Only will use 12 bits of resolution
        blocking
        """
        if start>end:
            for i in range(start,end):
                self.pca.channels[index].duty_cycle = i
                time.sleep(seconds/(start-end))
            self.logger.debug(f"Ramped up from {start} to {end} on index {index}")
        else:
            for i in reversed(range(start,end)):
                self.pca.channels[index].duty_cycle = i
                time.sleep(seconds/(end-start))
            self.logger.debug(f"Ramped down from {start} to {end} on index {index}")

    def stop(self,index:int):
        """takes the index from 0 to 15 and sets the duty cycle to 0 stopping it"""
        self.pca.channels[index].duty_cycle=0
        self.logger.debug(f"Stopped motor index {index}")

    def stopAll(self):
        """Stopps all of the motors by setting them to 0"""
        for i in range(0,self.numMotors-1):
            self.pca.channels[i].duty_cycle = 0
        self.logger.debug(f"Stopped all motors")

    def reset(self):
        """resets the PCA9685"""
        self.pca.reset()

    def digitalWrite(self,index:int,status:bool):
        """The digitalWrite function from arduino ported to PCA. 100% duty cycle or 0
        True is high false is low"""
        if status:
            self.pca.channels[index].duty_cycle = 0xffff
        else:
            self.pca.channels[index].duty_cycle = 0x0000

    def getChannels(self)->list:
        #returns all of the channels (pwm objects)
        return self.pca.channels
```

`motion.py`

```py
"""
Written By Yutaro U.
The piece of code that interfaces motion with the PCA
has a few classes
pip3 install adafruit-circuitpython-pca9685
https://docs.circuitpython.org/projects/pca9685/en/latest/

pip3 install adafruit-circuitpython-motor
https://docs.circuitpython.org/projects/motor/en/latest/
"""
import libs.movement.pca_interface as pca_interface
import logging
from typing import overload
from adafruit_motor import servo

interface = pca_interface.pca_backend()

class Motor:
    #initializer pca_index is the index of the PCA the id is the id of the motor
    #also need to git it pca_rot_pin_index which is the pin to hich the direction pin is attached
    def __init__(self,pca_index:int,pca_rot_pin_index:int,id:int,name:str=None) -> None:
        #set the duty to 0 when attaching for safety
        interface.setDutyCycle(pca_index,0)
        #set motor to forwad on default
        interface.digitalWrite(pca_index,False)
        self.pca_index = pca_index
        self.name = name
        self.id = id
        self.pca_rot_pin_index = pca_rot_pin_index
        self.logger = logging.getLogger()
        self.rotation = False
        self.currentPercent = 0

    def getName(self):
        return self.name

    def getId(self):
        return self.id

    #false is forward true is backwards
    def getRotation(self):
        return self.rotation

    def getSpeed(self):
        return self.currentPercent

    #set rotation False is forward True is backward
    def setRotation(self,rotation:bool):
        self.logger.debug(f"Set direction of motor id:{self.id} pcaidx:{self.pca_index} to {rotation}")
        interface.digitalWrite(self.pca_rot_pin_index,rotation)
        self.rotation = rotation

    #spin with no ramp up percents are in decimals from 0.0-1.0 false is forward trus is backward
    #spins at % speed not to % position
    def spin(self,percent:float,direction:bool=None):
        if percent>1:
            raise ValueError("Enter value between 0 and 1")
        if direction != None:
            self.setRotation(direction)
        self.currentPercent = percent
        interface.setDutyCycle(self.pca_index,int(0xFFFF*percent))
        self.logger.debug(f"Set direction of motor id:{self.id} pcaidx:{self.pca_index} to {percent*100}%")

    #ramp up/down the motor warning this function blocks. Do not increase seconds to more than 1 (may change later)
    def ramp(self,start:float,end:float,seconds:float=0.1):
        interface.rampDutyCycle(self.pca_index,start*0xFFFF,end*0xFFFF,seconds)

    def stop(self):
        interface.stop(self.pca_index)

class Servo:
    #Servo class, for the manipulator
    def __init__(self,pca_index:int,id:int,name:str=None) -> None:
        self.pca_index = pca_index
        self.name = name
        self.id = id
        self.logger = logging.getLogger()
        self.servo = servo.Servo(interface.getChannels()[pca_index])
        self.servo.angle=180

    def setAngle(self,angle:int)->None:
        self.servo.angle = angle

```

![Image of ROV by pool](~/assets/images/ROV/rov_at_pool.jpg)
This code allows me to easily create and attach motors and control them. In the actual code, we had a dictionary of `Motor` objects that corresponded to each motor on the ROV

### Matrix calculation code

Mr. S happened to be a exel wizard. He wrote up a microsoft excel spreadsheet which had some visual basic which allowed it to calculate how much each motor should contribute depending on which direction the vehicle wanted to move. I took the code and re-wrote it in python to generate a config file for the motors.<br>
`matrixCalc.py`

```py
#matrix calc for xbox input by Yutaro U. AKA yutarour
#The calculator for all things matrix involving ROV that utelizes matrixHelper

import libs.math_backend.matrixHelper as matrixHelper
#import matrixHelper
import numpy as np
import logging
import traceback
import json

default_matx_path = "./libs/math_backend/"
inverse_matx_name = "inverse_matrix_normal.npz"
maxes_name = "maxes.json"
computed_name = "computed.json"

def recompute(config_dir:str,save_dir:str=None):
    if save_dir == None:
        save_dir = default_matx_path

    matrixHelper.computeMotorParameters(config_dir,save_dir)

class matrixCalc():

    #set all of the variables needed for computing the matrix
    def __init__(self,maxes,inverse,computed):
        self.logger = logging.getLogger()
        #all of paths/objs were not defined, go look in the default dir
        if maxes == None and inverse == None and computed == None:
            try:
                #load inverse
                with open(default_matx_path+inverse_matx_name,'rb') as file:
                    self.inverse = np.load(file)
            except:
                self.logger.error("Failed to load Inverse\n"+traceback.format_exc(),stack_info=True)

            try:
                #load the maxes from json
                with open(default_matx_path+maxes_name,'r') as file:
                    self.maxes = json.load(file)
            except:
                self.logger.error("Failed to maxes\n"+traceback.format_exc(),stack_info=True)

            try:
                with open(default_matx_path+computed_name,'r') as file:
                    self.computed = json.load(file)

            except:
                self.logger.error("Failed to maxes\n"+traceback.format_exc(),stack_info=True)


        #if all of the stuff is passed in as parameters to dict
        if isinstance(maxes,dict) and isinstance(inverse,np.ndarray) and isinstance(computed,dict):
            self.inverse = inverse
            self.maxes = maxes
            self.computed = computed

        #if all of the args are paths
        if isinstance(maxes,str) and isinstance(inverse,str) and isinstance(computed,str):
            try:
                #load inverse
                with open(inverse,'rb') as file:
                    self.inverse = np.load(file)
            except:
                self.logger.error("Failed to load Inverse\n"+traceback.format_exc(),stack_info=True)

            try:
                #load the maxes from json
                with open(maxes,'r') as file:
                    self.maxes = json.load(file)
            except:
                self.logger.error("Failed to maxes\n"+traceback.format_exc(),stack_info=True)

            try:
                with open(computed,'r') as file:
                    self.computed = json.load(file)

            except:
                self.logger.error("Failed to maxes\n"+traceback.format_exc(),stack_info=True)


    def computeThrust(self,percents:list=None):
        #return nothing to do nothing for safety
        #x_trans,y_trans,z_trans,x_rot,y_rot,z_rot
        #receives as float between 0 and 1
        if percents == None:
            self.logger.warn("Percents were none in matrixCalc computeThrust")
            return None

        commands = np.matrix([
        [percents[0]*self.maxes["x_max"]],
        [percents[1]*self.maxes["y_max"]],
        [percents[2]*self.maxes["z_max"]],
        [percents[3]*self.maxes["x_r_max"]],
        [percents[4]*self.maxes["y_r_max"]],
        [percents[5]*self.maxes["z_r_max"]]
        ])

        return np.dot(self.inverse,commands)
```

`matrixHelper.py`

```py
"""
Matrix helper program written by Yutaro U. In colaboration with Mr. S
Used to calculate the thruster power for each thruster in order to move.
This is just the backend for calculating the inverse matrix and the max power parameters json doc
The movement percentages are calculated in matrixCalc.py which is in the same folder.
This requires a file called thruster_config.json to be located in
../motor_config.json
"""
import numpy as np
import math
import json

def computeMotorParameters(path,save_dir):
    thruster_data = {}
    with open(path,'r') as file:
        thruster_data = json.load(file)

    center_gravity = (thruster_data["center_gravity"]["x"],thruster_data["center_gravity"]["y"],thruster_data["center_gravity"]["z"])
    fwd_dir = thruster_data["forward_direction"]
    toe_in = thruster_data["toe_in"]
    thrust_elev = thruster_data["lat_thrust_elev_angle"]

    #define every row in the control matrix
    l_fx = []
    l_fy = []
    l_fz = []
    t_x = []
    t_y = []
    t_z = []
    for thruster in range(len(thruster_data["thrusters"])):
        #thruster_obj = thruster_data["thrusters"][thruster]
        #the capital ones are the ones calculated from the center of gravity
        thruster_data["thrusters"][thruster]["X"] = thruster_data["thrusters"][thruster]["x"]-center_gravity[0]
        thruster_data["thrusters"][thruster]["Y"] = thruster_data["thrusters"][thruster]["y"]-center_gravity[1]
        thruster_data["thrusters"][thruster]["Z"] = thruster_data["thrusters"][thruster]["z"]-center_gravity[2]

        if fwd_dir == "X":
            if thruster_data["thrusters"][thruster]["name"] == 0 or thruster ==thruster_data["thrusters"][thruster]["name"] == 2:
                thruster_data["thrusters"][thruster]["y-dir"] = math.sin(math.radians(toe_in))
            elif thruster_data["thrusters"][thruster]["name"] == 1 or thruster ==thruster_data["thrusters"][thruster]["name"] == 3:
                thruster_data["thrusters"][thruster]["y-dir"] = math.sin(math.radians(-toe_in))

            if thruster_data["thrusters"][thruster]["name"] != 4 and thruster_data["thrusters"][thruster]["name"] != 5:
                thruster_data["thrusters"][thruster]["x-dir"] = math.cos(math.radians(toe_in))
        #compute when forward dir not x
        else:
            if thruster_data["thrusters"][thruster]["name"] == 0 or thruster ==thruster_data["thrusters"][thruster]["name"] == 2:
                thruster_data["thrusters"][thruster]["x-dir"] = math.cos(math.radians(90-toe_in))
                thruster_data["thrusters"][thruster]["y-dir"] = math.sin(math.radians(90-toe_in))
            elif thruster_data["thrusters"][thruster]["name"] == 1 or thruster ==thruster_data["thrusters"][thruster]["name"] == 3:
                thruster_data["thrusters"][thruster]["x-dir"] = math.cos(math.radians(90+toe_in))
                thruster_data["thrusters"][thruster]["y-dir"] = math.sin(math.radians(90+toe_in))
            if thruster_data["thrusters"][thruster]["name"] != 4 and thruster_data["thrusters"][thruster]["name"] != 5:
                thruster_data["thrusters"][thruster]["x-dir"] = math.cos(math.radians(toe_in))
        #print(thruster)
        #compute z dirs
        if thruster_data["thrusters"][thruster]["name"] == 0 or thruster ==thruster_data["thrusters"][thruster]["name"] == 2:
            thruster_data["thrusters"][thruster]["z-dir"] = math.tan(math.radians(thrust_elev))
        if thruster_data["thrusters"][thruster]["name"] == 1 or thruster ==thruster_data["thrusters"][thruster]["name"] == 3:
            thruster_data["thrusters"][thruster]["z-dir"] = -1*math.tan(math.radians(thrust_elev))

        #compute vector lengths
        thruster_data["thrusters"][thruster]["vec_len"] = math.sqrt(math.pow(thruster_data["thrusters"][thruster]["x-dir"],2)+math.pow(thruster_data["thrusters"][thruster]["y-dir"],2)+math.pow(thruster_data["thrusters"][thruster]["z-dir"],2))

        #compute fx, fy, fz, tx, ty, tz
        thruster_data["thrusters"][thruster]["fx"] = thruster_data["thrusters"][thruster]["x-dir"]/thruster_data["thrusters"][thruster]["thrust"]*thruster_data["thrusters"][thruster]["vec_len"]
        thruster_data["thrusters"][thruster]["fy"] = thruster_data["thrusters"][thruster]["y-dir"]/thruster_data["thrusters"][thruster]["thrust"]*thruster_data["thrusters"][thruster]["vec_len"]
        thruster_data["thrusters"][thruster]["fz"] = thruster_data["thrusters"][thruster]["z-dir"]/thruster_data["thrusters"][thruster]["thrust"]*thruster_data["thrusters"][thruster]["vec_len"]

        #calculate rot
        thruster_data["thrusters"][thruster]["tx"] = (thruster_data["thrusters"][thruster]["Y"]*thruster_data["thrusters"][thruster]["fz"]-thruster_data["thrusters"][thruster]["Z"]*thruster_data["thrusters"][thruster]["fy"])/thruster_data["thrusters"][thruster]["vec_len"]
        thruster_data["thrusters"][thruster]["ty"] = (thruster_data["thrusters"][thruster]["Z"]*thruster_data["thrusters"][thruster]["fx"]-thruster_data["thrusters"][thruster]["X"]*thruster_data["thrusters"][thruster]["fz"])/thruster_data["thrusters"][thruster]["vec_len"]
        thruster_data["thrusters"][thruster]["tz"] = (thruster_data["thrusters"][thruster]["X"]*thruster_data["thrusters"][thruster]["fy"]-thruster_data["thrusters"][thruster]["Y"]*thruster_data["thrusters"][thruster]["fx"])/thruster_data["thrusters"][thruster]["vec_len"]
        #append values to the lists
        l_fx.append(thruster_data["thrusters"][thruster]["fx"])
        l_fy.append(thruster_data["thrusters"][thruster]["fy"])
        l_fz.append(thruster_data["thrusters"][thruster]["fz"])
        t_x.append(thruster_data["thrusters"][thruster]["tx"])
        t_y.append(thruster_data["thrusters"][thruster]["ty"])
        t_z.append(thruster_data["thrusters"][thruster]["tz"])
    #print(json.dumps(thruster_data,indent= 2))

    #np.linalg.inv()
    with open(save_dir+"computed.json",'w') as file:
        json.dump(thruster_data,file,indent=2)
    control_matrix = np.matrix([
        l_fx,
        l_fy,
        l_fz,
        t_x,
        t_y,
        t_z
    ])
    #pilot*inverse
    #save the inverse matrix
    print(control_matrix)
    print(np.linalg.det(control_matrix))
    inverse = np.linalg.inv(control_matrix)
    with open(save_dir+"inverse_matrix_normal.npz",'wb') as file:
        np.save(file,inverse)
        #cprint(np.linalg.inv(control_matrix))

    #sums
    x_trans = sum(l_fx)
    y_trans = sum(l_fy)
    z_trans = sum(l_fz)
    x_rot = sum(t_x)
    y_rot = sum(t_y)
    z_rot = sum(t_z)

    #get max value
    def gmv(input_list):
        ret_num = 0.0
        for num in input_list:
            if num>0:
                ret_num+=num
            else:
                ret_num+=abs(num)
        return ret_num
    x_max = gmv(l_fx)
    y_max = gmv(l_fy)
    z_max = gmv(l_fz)
    x_r_max = gmv(t_x)
    y_r_max = gmv(t_y)
    z_r_max = gmv(t_z)

    with open(save_dir+"maxes.json",'w') as file:
        json.dump({
            "x_max":x_max,
            "y_max":y_max,
            "z_max":z_max,
            "x_r_max":x_r_max,
            "y_r_max":y_r_max,
            "z_r_max":z_r_max
        },file,indent=2)


def testMotorCalc(percents:list = None):
    #this function is here for testing pourposes do not use it for actual implementations
    #needs to be executed in math_backend dir
    with open("maxes.json",'r') as file:
        maxes = json.load(file)
        x_max = maxes["x_max"]
        y_max = maxes["y_max"]
        z_max = maxes["z_max"]
        x_r_max = maxes["x_r_max"]
        y_r_max = maxes["y_r_max"]
        z_r_max = maxes["z_r_max"]

    if percents == None:
        percents = [100,100,100,0,0,0]

    with open("inverse_matrix_normal.npz",'rb') as file:
        inverse = np.load(file)

    commands = np.matrix([
        [percents[0]/100*x_max],
        [percents[1]/100*y_max],
        [percents[2]/100*z_max],
        [percents[3]/100*x_r_max],
        [percents[4]/100*y_r_max],
        [percents[5]/100*z_r_max]
    ])
    print(inverse.shape,commands.shape)
    #output the results of the calculation
    #print(inverse*commands)
    print(np.dot(inverse,commands))
    #identity
    #print(control_matrix*inverse)

    #multiplied =inverse*control_matrix
    #multiplied = multiplied[~np.all(multiplied == 0, axis=1)]
    #print(commands)
    #print(np.sum(multiplied))
    #print(control_matrix*multiplied)
    #print(multiplied)

#test func
#testMotorCalc([0,0,0,0,0,100])
```

`motor_config.json`

```json
{
  "lat_thrust_elev_angle": 1,
  "toe_in": 15,
  "forward_direction": "X",
  "center_gravity": {
    "x": 0,
    "y": 0,
    "z": 0
  },
  "thrusters": [
    {
      "thrust": 1,
      "name": 0,
      "x": 259,
      "y": -150,
      "z": -40
    },
    {
      "thrust": 1,
      "name": 1,
      "x": 258,
      "y": 150,
      "z": -40
    },
    {
      "thrust": -1,
      "name": 2,
      "x": -258,
      "y": 150,
      "z": -40
    },
    {
      "thrust": -1,
      "name": 3,
      "x": -259,
      "y": -150,
      "z": -40
    },
    {
      "thrust": 1,
      "name": 4,
      "x": -259,
      "y": 0,
      "z": 0,
      "x-dir": 0,
      "y-dir": 0,
      "z-dir": 1
    },
    {
      "thrust": 1,
      "name": 5,
      "x": 259,
      "y": 0,
      "z": 0,
      "x-dir": 0,
      "y-dir": 0,
      "z-dir": 1
    }
  ]
}
```

Yeah... This was overkill in so many aspects...

### Custom PCB design

I also made a custom PCB to power all of the motor drivers and the raspberry pi board through a buck converter.
![PDB PCB](~/assets/images/ROV/PDB_PCB.png)
This 4 layer abomination was probably my 3rd PCB. (to learn more about what I've been doing in terms of PCB design, look at this article [PCB Design]()). I didn't know what a ground plane was at that time. The crazy thing is that it still worked.

## How it went

On the day of the competition, we showed up, but during the test time, we realized that the ROV was acting weird. The technical debt from overcomplicating the thrust calculations bit us in the back. During the actual competition, the ROV acted out as well by rebooting every 10 seconds or so. We were not able to do that well.
<br><br>
This rebooting behavior was caused by the competition's power supply's safety feature activating when it sensed a sudden spike in current draw. We had not encountered this problem as we were using a very robust power supply compared to the one the competition had.
![Power supply image](~/assets/images/ROV/power_supply.jpg)
Above: The power supply we were using.

## What I Learned/What I Could Have Done Better

This experience has undoubtedly taught me a lot about engineering and working with others. The main thing I learned is that as a leader, delegating work to others is very important. Them not knowing how to do something is not an excuse to take all the work and do it yourself. That's just not efficient. <br><br>
Another thing I learned is that when you don’t have a way to sift through talent, and get a wide range of people, you must learn how to work with the resources you have. I probably had to be more proactive when it came to telling people to learn about certain technologies and ways of doing something.
