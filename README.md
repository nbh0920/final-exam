# final-exam
from IO7FuPython import ConfiguredDevice
import uComMgr32
import json
import time
from machine import ADC, Pin, PWM


def handleCommand(topic, msg):
    pass

def handleMeta(topic, msg):
    pass


nic = uComMgr32.startWiFi('IoT518')

device = ConfiguredDevice()
device.setUserCommand(handleCommand)
device.setUserMeta(handleMeta)
device.connect()



# MQ135 공기질 센서
mq = ADC(Pin(1))
mq.atten(ADC.ATTN_11DB)
mq.width(ADC.WIDTH_12BIT)

# 빗물 센서 (AO 입력 → GPIO2 : ADC2)
rain_ao = ADC(Pin(2))
rain_ao.atten(ADC.ATTN_11DB)
rain_ao.width(ADC.WIDTH_12BIT)

# 빗물 센서 (DO 입력 → GPIO10 : 디지털)
rain_do = Pin(10, Pin.IN)

# DO 기준값: 젖으면 LOW(0), 마르면 HIGH(1)
RAIN_WET = 0


servo = PWM(Pin(13), freq=50)
current_angle = 180

def set_angle(angle):
    global current_angle
    duty = int((angle / 180) * 5000 + 1000)
    servo.duty_u16(duty)
    current_angle = angle

def move_servo(target_angle, duration_ms):
    global current_angle
    start = current_angle
    end = target_angle
    if start == end:
        return

    step = 1 if end > start else -1
    steps = abs(end - start)
    delay = duration_ms / steps / 1000

    angle = start
    for _ in range(steps):
        angle += step
        set_angle(angle)
        time.sleep(delay)

OPEN_ANGLE = 0
CLOSE_ANGLE = 135
OPEN_TIME_MS = 2000
CLOSE_TIME_MS = 1500

window_state = "CLOSED"
set_angle(CLOSE_ANGLE)


GOOD_THRESHOLD = 720   # 이하면 '공기 좋음'으로 보고 OPEN
BAD_THRESHOLD  = 721   # 이상이면 '공기 나쁨'으로 보고 CLOSED



# 메인 루프
while True:
    
    air_value = mq.read()
    rain_value_ao = rain_ao.read()
    rain_value_do = rain_do.value()  # 0=젖음, 1=마름

    print("Air:", air_value,
          " RainAO:", rain_value_ao,
          " RainDO:", rain_value_do,
          " State:", window_state)
    

    # DO 기준으로 비 감지
    if rain_value_do == RAIN_WET:
        if window_state != "CLOSED":
            print("비 감지됨 → 창문 닫음")
            move_servo(CLOSE_ANGLE, CLOSE_TIME_MS)
            window_state = "CLOSED"

    else:
        # 공기질 조건
        if air_value <= GOOD_THRESHOLD and window_state != "OPEN":
            print("외부 공기 좋음 → 창문 염")
            move_servo(OPEN_ANGLE, OPEN_TIME_MS)
            window_state = "OPEN"

        elif air_value >= BAD_THRESHOLD and window_state != "CLOSED":
            print("외부 공기 나쁨 → 창문 닫음")
            move_servo(CLOSE_ANGLE, CLOSE_TIME_MS)
            window_state = "CLOSED"

    
    # 서버로 값 전송
    status_data = {
        'd': {
            'air': air_value,
            'rain_ao': rain_value_ao,
            'rain_do': rain_value_do,
            'window': window_state
        }
    }
    device.publishEvent('status', json.dumps(status_data))

    time.sleep(6)

