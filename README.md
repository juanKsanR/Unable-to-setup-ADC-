# Unable-to-setup-ADC-
# I have this problema with Beaglebone. I have a screen 3,5" and I use the AIN ports for 5 sensors. WHAT DO I DO TO SOLVE THIS?


import Adafruit_BBIO.ADC as ADC
import Adafruit_BBIO.GPIO as GPIO
import time
import sys
import pyautogui
from PyQt5.QtCore import *
from PyQt5 import uic, QtWidgets
from time import strftime

ADC.setup()

GPIO.setup("P8_19", GPIO.IN) #botoneras
GPIO.setup("P8_20", GPIO.IN)
GPIO.setup("P8_21", GPIO.IN)
GPIO.setup("P8_22", GPIO.IN)

GPIO.setup("P8_3", GPIO.OUT) #luz MODO MANUAL
GPIO.setup("P8_4", GPIO.OUT) #luz MODO AUTO
GPIO.setup("P8_5", GPIO.OUT) #alarma luz
GPIO.setup("P8_6", GPIO.IN) #alarma in

GPIO.setup("P8_13", GPIO.OUT) #alarma sonora FC100 al max
GPIO.setup("P8_25", GPIO.OUT)
GPIO.setup("P8_26", GPIO.OUT)


qtCreatorFile = "/home/debian/etapa2_1/hidro1.ui"

Ui_MainWindow, QtBaseClass = uic.loadUiType(qtCreatorFile)

# Variables Globales 
sensor_ft200 = 0.0
sensor_pt201 = 0.0
sensor_pt202 = 0.0
sensor_pt203 = 0.0
pdr = 2.0
pdr_set = 2.0

class hidrociclon1(QtWidgets.QMainWindow, Ui_MainWindow):
	def __init__(self):
		QtWidgets.QMainWindow.__init__(self)
		Ui_MainWindow.__init__(self)
		self.setupUi(self)
		self.set_inicial()
		self.estados()

	def fechas(self):
		dia=strftime("%d/%b/%Y  %H:%M")
		self.fechap.setText(dia)

	def set_inicial(self):
		self.set1.setValue(600)
		self.set2.setValue(95)
		self.set3.setValue(10)
		self.set4.setValue(45)
		self.set5.setValue(100)
		self.set6.setValue(0)
		self.set6_1.setValue(0.1)
		self.set6_2.setValue(2.0)

		self.estado.setCurrentText("MANUAL")

		self.alarm.setText("ALARMA OFF")
		self.alarm.setStyleSheet("background-color: gray; color: black")

		GPIO.output("P8_5", GPIO.LOW) # alarma boton

	def rotacion(self):
		if GPIO.input("P8_19"):
			pyautogui.press('tab')
		if GPIO.input("P8_21"):
			pyautogui.press('up')
		if GPIO.input("P8_22"):
			pyautogui.press('down')
		if GPIO.input("P8_20"):
			pyautogui.press('tab')
			pyautogui.press('tab')
			pyautogui.press('tab')
			pyautogui.press('tab')

	def estados(self):
		self.fechas()

		if self.estado.currentText() == "MANUAL":
			if GPIO.input("P8_6"):
				self.emergencia()
			else:
				self.estado.setStyleSheet("background-color: green")
				GPIO.output("P8_3", GPIO.HIGH)
				GPIO.output("P8_4", GPIO.LOW)
				GPIO.output("P8_5", GPIO.LOW)
				self.alarm.setText("ALARMA OFF")
				self.alarm.setStyleSheet("background-color: gray; color: black")

				self.sensores()

				fc200 = self.set5.value()
				if fc200 == 0:
					GPIO.output("P8_25", GPIO.LOW)
					self.act1.setText("0")
				else:
					GPIO.output("P8_25", GPIO.HIGH)
					self.act1.setText(str(fc200))

				pc200 = self.set6.value()
				if pc200 == 0:
					GPIO.output("P8_26", GPIO.LOW)
					self.act2.setText("0")
				else:
					GPIO.output("P8_26", GPIO.HIGH)
					self.act2.setText(str(pc200))

		else:
			if GPIO.input("P8_6"):
				self.emergencia()
				GPIO.output("P8_5", GPIO.HIGH)
			else:
				GPIO.output("P8_3", GPIO.LOW)
				GPIO.output("P8_4", GPIO.HIGH)
				self.estado.setStyleSheet("background-color: blue")

				self.sensores()
				self.lazo_200_50_1()
				self.lazo_200_50_2()

		QTimer.singleShot(500, self.estados)

	def emergencia(self):
		GPIO.output("P8_5", GPIO.HIGH)
		self.alarm.setText("ALARMA ON")
		self.alarm.setStyleSheet("background-color: red; color: white")


	def sensores(self):
		global sensor_ft200
		global sensor_pt201
		global sensor_pt202
		global sensor_pt203

		lec1 = ADC.read("P9_33")
		sensor_ft200 =int(lec1*1333.333)
		self.sensor1.setText(str(sensor_ft200))

		lec2 = ADC.read("P9_35")
		sensor_pt201 = int(lec2*133.333)
		self.sensor2.setText(str(sensor_pt201))

		lec3 = ADC.read("P9_36")
		sensor_pt202 =int(lec3*133.333)
		self.sensor3.setText(str(sensor_pt202))

		lec4 = ADC.read("P9_37")
		sensor_pt203 =int(lec4*133.333)
		self.sensor4.setText(str(sensor_pt203))


	def lazo_200_50_1(self):

		global sensor_ft200

		global pdr
		global pdr_set

		print("--------------------------")
		print("-- LAZO 1 --")

		GPIO.output("P8_5", GPIO.LOW)
		self.alarm.setText("ALARMA OFF")
		self.alarm.setStyleSheet("background-color: gray; color: black")

		tolerancia = 0.2
		if ((pdr_set - tolerancia) <= pdr) and ((pdr_set + tolerancia) >= pdr):

			print("Ejecuta el lazo 1")
			set_ft200 = self.set1.value()
			apertura_fc200 = self.set5.value()

			error_ft200 = set_ft200 - sensor_ft200
			kp = 0.01
			out1 = round(kp * error_ft200)

			apertura_fc200 += out1

			if apertura_fc200 < 0:
				apertura_fc200 = 0

			if apertura_fc200 > 100:
				apertura_fc200 = 100
			self.set5.setValue(apertura_fc200)
			self.act1.setText(str(apertura_fc200))
		else:
			print("Prioridad lazo 2")


	def lazo_200_50_2(self):

		print("--------------------------------")
		print("-- LAZO 2 --")

		global sensor_pt201
		global sensor_pt202
		global sensor_pt203
		global pdr
		global pdr_set

		print("PT 201",sensor_pt201)
		print("PT 202",sensor_pt202)
		print("PT 203",sensor_pt203)

		GPIO.output("P8_5", GPIO.LOW)
		self.alarm.setText("ALARMA OFF")
		self.alarm.setStyleSheet("background-color: gray; color: black")

		apertura_pc200 = self.set6.value()

		k = self.set6_1.value()
		pdr_set = self.set6_2.value()

		if sensor_pt203 == sensor_pt201:
			sensor_pt203 = sensor_pt201 - 1

		pdr = round((sensor_pt201 - (k*sensor_pt203)) / (sensor_pt201 - sensor_pt203),1)
		print("PDR medido",pdr)
		print("PDR set   ",pdr_set)

		# Constante de proporcionalidad para la correccion
		kp = 1
		errorPDR = round(pdr - pdr_set,1)
		print("Error del PDR",errorPDR)
		ajusteValvula = kp * errorPDR
		print("Ajuste a la valvula",ajusteValvula)

		# Abre la valvula para bajar el PDR
		# Cierra la valvula para subir el PDR

		apertura_pc200 += ajusteValvula

		if apertura_pc200 > 99:
			apertura_pc200 = 100

		if apertura_pc200 < 1:
			apertura_pc200 = 0

		print("Valvula abierta a",apertura_pc200)
		self.set6.setValue(apertura_pc200)
		self.act2.setText(str(int(apertura_pc200)))
		self.act2_2.setText(str(pdr))





if __name__ == "__main__":
	app =  QtWidgets.QApplication(sys.argv)
	window = hidrociclon1()
	window.show()
	sys.exit(app.exec_())
