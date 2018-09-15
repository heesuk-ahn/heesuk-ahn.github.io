---
layout: post
title: 1. Python을 이용하여 라즈베리파이 gpio를 제어해보자.
category: Python
tags: [python]
---

오늘은 Python을 이용해서 라즈베리파이의 gpio를 제어해보려고 합니다.

gpio는 라즈베리파이의 제어할 수 있는 핀들인데, 이 핀들을 센서와 연결하고, 해당 센서로부터 input 되는

값을 이용하여 google iot core에 publish 해보려고 합니다.

##Python Gpio 라이브러리를 다운로드 받자.

파이썬에서 Gpio 라이브러리는 `WiringPi`와 `RPI.Gpio`가 있는데, 이 포스트에서는 `RPI.Gpio`를 이용해보려고 합니다.

RPI.GPIO 라이브러리에 대한 상세설명은 `https://pypi.org/project/RPi.GPIO/`에서 확인이 가능합니다.

python과 pip이 설치되어 있다면, `pip install RPi.GPIO` 커맨드를 이용해서 간단하게 라이브러리를 받아올 수 있습니다.

단, Gpio는 라즈베리파이 모듈이므로, 라즈베리파이의 OS 환경에서 받는 것이 아니라면, 에러가 발생할 수 있습니다.

 저의 경우, 제 PC의 환경이 Mac OS이고, Mac OS 환경에서 Pycharm IDE를 이용해서 코딩을 하고 싶어서

`RPi.GPIO` 가짜 라이브러리를 마치 모킹하듯이 만들어서 진행하려고 합니다.

물론, 실제 라즈베리 환경에서는 Python GPIO 라이브러리를 다운로드 받아서 쓸 생각입니다.

실제 여러분들의 환경이 만약 저와 같이 다른 OS 환경에서 코딩 후에 Github에 올린 후, 라즈베리파이에서

Git을 이용해서 가져오실 예정이라면, 제가 하는 방법이 도움이 될 수 있을 것 같습니다.

 물론, 실제로 라즈베리파이 환경에서 코딩을 하시는 분들의 경우에도 코드상으로는 같은 코드이기 때문에

 아래 코드 예제들을 따라해주시면 되겠습니다.

## Rpi.Gpio의 기본 함수들을 알아보고, Config 설정을 해보자.

###- GPIO.setmode

  라즈베리파이는 여러 버젼들이 있습니다. 구버젼의 라즈베리파이는 26핀이고, B+ 이상의 버젼은 40핀입니다.
  즉, 하드웨어의 핀 구조가 다르기 때문에 GPIO 라이브러리에서는 어떤 버젼인지에 따라서 mode를 설정해줄 수 있습니다.
  mode에는 2가지가 있는데, 구버젼의 경우는 `BOARD`를 사용하고, 신버젼의 경우에는 `BCM` 모드를 사용합니다.

  ```
   만약, 구버젼 라즈베리파이일 경우

      `GPIO.setmode(GPIO.BOARD)`

   B+ 이상의 라즈베리파이일 경우,

      `GPIO.setmode(GPIO.BCM)`    
  ```
###- GPIO.getmode

  만약, 시스템 레벨에서 어떤 mode인지를 가져오기 위해서는 getmode 함수를 사용하시면 됩ㄴ디ㅏ.

  ```
    mode = GPIO.getmode()

  ```  
  이때, mode는 optional한 값이며, 그 값은 `GPIO.BOARD` 또는 `GPIO.BCM` 또는 None 값일 수 있습니다.

###- GPIO.setup

  setup 함수의 경우, GPIO의 핀들에 대한 설정을 해준다고 생각하시면 됩니다. 센서로부터 값이 input이 되는 경우가 있고,
  아니면 라즈베리파이보드에서 => 센서로 out이 되는 경우가 있습니다.
  간단하게 예를 들면, 만약 습도 센서로부터 데이터 값을 입력받을 건데, 그 핀 번호를 14번 핀으로 부터 input을 받겠다 라고
  한다면,

   `GPIO.setup(14, GPIO.IN)`

  위와 같이 사용하실 수 있으며, 반대로, 어떤 릴레이 모듈의 전원을 on / off 제어를 해야해서 라즈베리파이에서 센서로 output 이벤트를
  줘야하는 경우에는

   `GPIO.setup(14, GPIO.OUT)`

  위와 같이 설정을 할 수 있습니다.

  만약, 여러 채널을 동시에 등록하고 싶다면, RPI.Gpio는 이것 또한 지원을 해주는데,

  ```
  chan_list = [11,12]    # add as many channels as you want!
                       # you can tuples instead i.e.:
                       #   chan_list = (11,12)
  GPIO.setup(chan_list, GPIO.OUT)
  ```
  위와 같이 채널을 튜플 형태나, 어레이 형태로 넣을 수도 있습니다.

  이러한 setup은 `채널을 개통한다`라고 생각하시면 됩니다.

###- GPIO.input

  이제 실제로 설정한 채널로 부터 센서값을 받는다고 해봅시다. 센서값을 읽어들이기 위해서

  ```
  GPIO.input(channel)
  ```
  를 이용하여서 데이터 값을 읽어들일 수 있습니다. 데이터 값을 읽어들였을 때, 돌아올 수 있는 return 값은

  `0 / GPIO.LOW / False or 1 / GPIO.HIGH / True.` 이 있습니다.

###- GPIO.output


###- GPIO.cleanup

 GPIO 핀을 설정 후에, 만약 프로그램을 종료한다고 했을 때, 설정된 pin 값들에 대한 정보를 모두 초기화 해주어야 합니다.
 그렇지 않을경우, 다시 실행할 때, 경고 메세지가 나오게 됩니다.

 `except KeyboardInterrupt:
    GPIO.cleanup()`

 위와 같이 키보드 인터럽트를 설정해 주시고, 실제 키보드에서 ctrl+c 커맨드를 통해서 종료시에 위에 함수가 실행되며 설정이 초기화 됩니다.    

## [Mocking!] 위의 GPIO 함수들을 `가짜`로 만들어보자.

 위에서 제가 한번 말한 것처럼, GPIO 라이브러리를 OS X에서 직접적으로 사용할 수 없으므로, `가짜` 함수들을 만들어보려고 합니다.

 `만약, 이미 라즈베리파이 환경이고, GPIO 라이브러리를 받으셨다면, 이 부분은 패스하셔도 됩니다.`


 * 참조 : https://sourceforge.net/p/raspberry-gpio-python/wiki/BasicUsage/ : python gpio library documentation
         https://kocoafab.cc/tutorial/view/310 : GPIO핀을 이용하여 LED제어해보기 by kocoafab
