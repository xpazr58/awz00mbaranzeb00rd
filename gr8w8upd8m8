#!/usr/bin/env python

import collections
import time
import bluetooth
import sys
import subprocess
import binascii

CONTINUOUS_REPORTING = b"\x04"  # Easier as string with leading zero

COMMAND_LIGHT = b"\x11"
COMMAND_REPORTING = b"\x12"
COMMAND_REQUEST_STATUS = b"\x15"
COMMAND_REGISTER = b"\x16"
COMMAND_READ_REGISTER = b"\x17"

#input is Wii device to host
INPUT_STATUS = b"\x20"
INPUT_READ_DATA = b"\x21"

EXTENSION_8BYTES = b"\x32"
#end "hex" values
# [!]
# Refer to line 237 - 241
# [!]

BUTTON_DOWN_MASK = 8

TOP_RIGHT = 0
BOTTOM_RIGHT = 1
TOP_LEFT = 2
BOTTOM_LEFT = 3


BLUETOOTH_NAME = "Nintendo RVL-WBC-01"


class EventProcessor:
	def __init__(self):
		self._measured = False
		self.done = False
		self._events = []

	def mass(self, event):
		if event.totalWeight > 30:
			self._events.append(event.totalWeight)
			if not self._measured:
				print("Starting measurement.")
				self._measured = True
		elif self._measured:
			self.done = True

	@property
	def weight(self):
		if not self._events:
			return 0
		histogram = collections.Counter(round(num, 1) for num in self._events)
		return histogram.most_common(1)[0][0]


class BoardEvent:
	def __init__(self, topLeft, topRight, bottomLeft, bottomRight, buttonPressed, buttonReleased):

		self.topLeft = topLeft
		self.topRight = topRight
		self.bottomLeft = bottomLeft
		self.bottomRight = bottomRight
		self.buttonPressed = buttonPressed
		self.buttonReleased = buttonReleased
		#convenience value
		self.totalWeight = topLeft + topRight + bottomLeft + bottomRight


class Wiiboard:
	def __init__(self, processor):
		# Sockets and status
		self.receivesocket = None
		self.controlsocket = None

		self.processor = processor
		self.calibration = []
		self.calibrationRequested = False
		self.LED = False
		self.address = None
		self.buttonDown = False
		for i in range(3):
			self.calibration.append([])
			for j in range(4):
				self.calibration[i].append(10000)  # high dummy value so events with it don't register

		self.status = "Disconnected"
		self.lastEvent = BoardEvent(0, 0, 0, 0, False, False)

		try:
			self.receivesocket = bluetooth.BluetoothSocket(bluetooth.L2CAP)
			self.controlsocket = bluetooth.BluetoothSocket(bluetooth.L2CAP)
		except ValueError:
			raise Exception("Error: Bluetooth not found")

	def isConnected(self):
		return self.status == "Connected"

	# Connect to the Wiiboard at bluetooth address <address>
	def connect(self, address):
		if address is None:
			print("Non existant address")
			return
		self.receivesocket.connect((address, 0x13))
		self.controlsocket.connect((address, 0x11))
		if self.receivesocket and self.controlsocket:
			print("Connected to Wiiboard at address " + address)
			self.status = "Connected"
			self.address = address
			self.calibrate()
			self.send(COMMAND_REGISTER, b"\x04\xA4\x00\x40\x00")
			self.setReportingType()
			print("Wiiboard connected")
		else:
			print("Could not connect to Wiiboard at address " + address)

	def receive(self):
		#try:
		#   self.receivesocket.settimeout(0.1)       #not for windows?
		while self.status == "Connected" and not self.processor.done:
			data = self.receivesocket.recv(25)
			intype = hex(int(data[1]))
			if intype == "0x20":
				# TODO: Status input received. It just tells us battery life really
				self.setReportingType()
			elif intype == "0x21":
				if self.calibrationRequested:
					packetLength = int(hex(int(data[4])), 16) / 16 + 1
					self.parseCalibrationResponse(data)

					if packetLength < 16:
						self.calibrationRequested = False
			elif intype == "0x32":
				self.processor.mass(self.createBoardEvent(data[2:12]))
			else:
				print("ACK to data write received")

		self.status = "Disconnected"
		self.disconnect()

	def disconnect(self):
		if self.status == "Connected":
			self.status = "Disconnecting"
			while self.status == "Disconnecting":
				self.wait(100)
		try:
			self.receivesocket.close()
		except:
			pass
		try:
			self.controlsocket.close()
		except:
			pass
		print("WiiBoard disconnected")

	# Try to discover a Wiiboard
	def discover(self):
		print("Press the red (or black) sync button on the board !!")
		address = None
		bluetoothdevices = bluetooth.discover_devices(duration=3, lookup_names=True)
		for bluetoothdevice in bluetoothdevices:
			if bluetoothdevice[1] == BLUETOOTH_NAME:
				address = bluetoothdevice[0]
				print("Found Wiiboard at address " + address)
		if address is None:
			print("No Wiiboards discovered.")
		return address

	def createBoardEvent(self, byte):
		buttonBytes = byte[0:2]
		byte = byte[2:len(byte)-12]
		buttonPressed = False
		buttonReleased = False

		state = (int(str(buttonBytes[0]), 16) << 8) | int(str(buttonBytes[1]), 16)
		if state == BUTTON_DOWN_MASK:
			buttonPressed = True
			if not self.buttonDown:
				print("Button pressed")
				self.buttonDown = True

		if not buttonPressed:
			if self.lastEvent.buttonPressed:
				buttonReleased = True
				self.buttonDown = False
				print("Button released")

		rawTR = (int(str(byte[0]), 16) << 8)
		rawBR = (int(str(byte[1]), 16) << 8)
		rawTL = (int(str(byte[2]), 16) << 8)
		rawBL = (int(str(byte[3]), 16) << 8)

		topLeft = self.calcMass(rawTL, TOP_LEFT)
		topRight = self.calcMass(rawTR, TOP_RIGHT)
		bottomLeft = self.calcMass(rawBL, BOTTOM_LEFT)
		bottomRight = self.calcMass(rawBR, BOTTOM_RIGHT)
		boardEvent = BoardEvent(topLeft, topRight, bottomLeft, bottomRight, buttonPressed, buttonReleased)
		return boardEvent

	def calcMass(self, raw, pos):
		val = 0.0
		#calibration[0] is calibration values for 0kg
		#calibration[1] is calibration values for 17kg
		#calibration[2] is calibration values for 34kg
		if raw < int(self.calibration[0][pos], 16):
			return val
		elif raw < int(self.calibration[1][pos], 16):
			val = 17 * ((raw - int(self.calibration[0][pos], 16)) / float((int(self.calibration[1][pos], 16) - int(self.calibration[0][pos], 16))))
		elif raw > int(self.calibration[1][pos], 16):
			val = 17 + 17 * ((raw - int(self.calibration[1][pos], 16)) / float((int(self.calibration[2][pos], 16) - int(self.calibration[1][pos], 16))))

		return val

	def getEvent(self):
		return self.lastEvent

	def getLED(self):
		return self.LED

	def parseCalibrationResponse(self, data):
#		index = 0
#		byte = str(byte.replace("\\x", ""))
#		if len(byte) == 16:
#			for i in range(2):
#				for j in range(4):
#					self.calibration[i][j] = (int(byte[index], 16) << 8) + int(byte[index + 1], 16)
#					index += 2
#		elif len(byte) < 16:
#			for i in range(4):
#				self.calibration[2][i] = (int(byte[index], 16) << 8) + int(byte[index + 1], 16)
#				index += 2
		length = int(data[4] / 16 + 1)
		data = data[7:7 + length]
		cal = lambda d: [d[j:j+2].hex() for j in [0, 2, 4, 6]]
		if length == 16: # First packet of calibration data
			self.calibration = [cal(data[0:8]), cal(data[8:16]), [1e4]*4]
		elif length < 16: # Second packet of calibration data
			self.calibration[2] = cal(data[0:8])

	# Send <data> to the Wiiboard
	# [!]
	# Revised send function (from PierrickKoch/wiiboard)
	# Now instead of storing them in variables you DIRECTLY send them
	# Read below :)
	# [1]
	def send(self, *data):
		if self.status != "Connected":
			return
		self.controlsocket.send(b'\x52'+b''.join(data))
	#Turns the power button LED on if light is True, off if False
	#The board must be connected in order to set the light
	def setLight(self, light):
		if light:
			val = b"\x10"
		else:
			val = b"\x00"

		self.send(COMMAND_LIGHT, val)
		self.LED = light

	def calibrate(self):
		self.send(COMMAND_READ_REGISTER, b"\x04\xA4\x00\x24\x00\x18")
		self.calibrationRequested = True

	def setReportingType(self):
		self.send(COMMAND_REPORTING, CONTINUOUS_REPORTING, EXTENSION_8BYTES)

	def wait(self, millis):
		time.sleep(millis / 1000.0)


def main():
	processor = EventProcessor()

	board = Wiiboard(processor)
	if len(sys.argv) == 1:
		print("Discovering board...")
		address = board.discover()
	else:
		address = sys.argv[1]

	try:
		# Disconnect already-connected devices.
		# This is basically Linux black magic just to get the thing to work.
		subprocess.check_output(["bluez-test-input", "disconnect", address], stderr=subprocess.STDOUT)
		subprocess.check_output(["bluez-test-input", "disconnect", address], stderr=subprocess.STDOUT)
	except:
		pass

	print("Trying to connect...")
	board.connect(address)  # The wii board must be in sync mode at this time
	board.wait(200)
	# Flash the LED so we know we can step on.
	board.setLight(False)
	board.wait(500)
	board.setLight(True)
	board.receive()

	print(processor.weight)

	# Disconnect the balance board after exiting.
	subprocess.check_output(["bluez-test-device", "disconnect", address])

if __name__ == "__main__":
	main()
