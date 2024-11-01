# ultimo-lab
## Materiales
Los materiales implementados para la realización de este laboratorio fueron esenciales para el correcto funcionamiento los cuales serán mencionados a continuación:
- AD8232
- Electrodos
- Cables Para el sensor AD8292
- Jumpers macho hembra
- Arduino
- Computador
- Python

## Introducción 
Para el Desarrollo de esta práctica de laboratorio se tuvo en cuenta algunos factores teóricos con el fin, de llevar a cabo el análisis de una señal ECG. Para ello, se tuvo cuenta algunos conceptos como el sistema simpático y parasimpático y sus repercusiones en el comportamiento de la frecuencia cardiaca y su variabilidad en el intervalo R-R de una señal biológica. Una vez conociendo estos conceptos teóricos se procede y obtenido la señal ECG por medio del sensor AD8232 se hace uso de la señal aplicando la transformada de Wavelet específicamente la de Morlet y otros parámetros de obtención de información de la señal ECG.
Procedimiento
- Autorización: Para el desarrollo de esta prueba se realizó una autorización al paciente para el tratamiento y uso de los datos personales del paciente.
- Adquisición Señal: Por medio de la conexión del sensor AD8232 al paciente se realiza la conversión de análogo a digital por medio del microcontrolador Arduino, y posteriormente se realiza el procesamiento de la señal en el computador por medio de Python
- Procesamiento de la señal: Para este laboratorio se aplica algunos conceptos como transformada de Wavelet (Morlet), espectros de frecuencia conocer los picos de la señal para determinar las distancias R-R
- Análisis: Entender los valores y funcionamiento de todos los parámetros vistos y aplicados.	

## Diclaimer:
Este documento se proporciona de forma libre para su uso educativo. Si deseas utilizar este material con otros fines, por favor, comunícate conmigo para obtener autorización.
#### Código Adquisición de la señal en Python

	import serial
	import time
	import numpy as np

	# Configura el puerto serial
	arduino = serial.Serial('COM5', 9600)  # Ajusta 'COM5' según el puerto correcto
	time.sleep(2)  # Espera a que el puerto se inicialice

	data = []
	start_time = time.time()
	duration = 10  # Duración en segundos (5 minutos)

	while time.time() - start_time < duration:
		if arduino.in_waiting:
			value = arduino.readline().decode('utf-8').strip()
			if value.isdigit():
				data.append(int(value))

	arduino.close()

	# Guardar la señal en un archivo
	filename = 'signal_data.npy'
	np.save(filename, np.array(data))

	print(f'Datos guardados en {filename}')
[![image.png](https://i.postimg.cc/kMxnPmK1/image.png)](https://postimg.cc/JtzwjSTZ)
 
#### Código para Programar la adquisición en Arduino
	int sensorPin = A0; // Pin del sensor
	int sensorValue = 0; 

	void setup() {
	  Serial.begin(9600);
	}

	void loop() {
	  sensorValue = analogRead(sensorPin); // Leer valor del sensor
	  Serial.println(sensorValue);
	  delay(100); // Retardo para hacer más legible la salida
	}



#### Código picos de la señal

	import numpy as np
	import matplotlib.pyplot as plt
	from scipy.signal import find_peaks, butter, filtfilt

	# Parámetros de la señal ECG
	frecuencia_cardiaca = 70  # Frecuencia cardíaca promedio en latidos por minuto
	duracion = 300  # Duración en segundos (5 minutos)
	frecuencia_muestreo = 250  # Frecuencia de muestreo en Hz (250 muestras por segundo)
	duracion_analisis = 20  # Duración en segundos para el análisis (20 segundos)

	# Tiempo de la señal
	t = np.linspace(0, duracion, duracion * frecuencia_muestreo)
	t_analisis = np.linspace(0, duracion_analisis, duracion_analisis * frecuencia_muestreo)

	# Simulación de una señal ECG con picos R-R y un poco más de ruido
	def simulate_ecg(t, frecuencia_cardiaca, frecuencia_muestreo):
		ecg_signal = np.zeros_like(t)
		rr_interval = int((60 / frecuencia_cardiaca) * frecuencia_muestreo)
		for i in range(0, len(t), rr_interval):
			if i + 5 < len(t):
				ecg_signal[i:i+5] = 1  # Pico R simple
			ecg_signal += 0.01 * np.random.randn(len(t))  # Añadiendo ruido blanco
			if np.random.rand() > 0.98:  # Introduce variaciones aleatorias
				ecg_signal[i:i+5] = 1.2 * ecg_signal[i:i+5]  # Pico más alto
		return ecg_signal

	ecg_signal = simulate_ecg(t, frecuencia_cardiaca, frecuencia_muestreo)

	# Filtrado para imitar las características del AD8232 (simple filtro pasa bajo)
	def butter_lowpass_filter(data, cutoff, fs, order=5):
		nyq = 0.5 * fs
		normal_cutoff = cutoff / nyq
		b, a = butter(order, normal_cutoff, btype='low', analog=False)
		y = filtfilt(b, a, data)
		return y

	cutoff = 50  # Frecuencia de corte en Hz
	ecg_filtered = butter_lowpass_filter(ecg_signal, cutoff, frecuencia_muestreo)

	# Guardar la señal en un archivo
	filename = 'señal_real.npy'
	np.save(filename, ecg_signal)

	# Analizar solo los primeros 20 segundos
	ecg_filtered_20s = ecg_filtered[:duracion_analisis * frecuencia_muestreo]

	# Identificar los picos R en los primeros 20 segundos
	peaks, _ = find_peaks(ecg_filtered_20s, distance=frecuencia_muestreo * 0.6, height=np.mean(ecg_filtered_20s))

	# Calcular los intervalos R-R en los primeros 20 segundos
	rr_intervals_20s = np.diff(peaks) / frecuencia_muestreo

	# Graficar la señal filtrada y los picos R identificados en los primeros 20 segundos
	plt.figure(figsize=(12, 6))
	plt.plot(t_analisis, ecg_filtered_20s, label='Señal Filtrada (20s)')
	plt.plot(peaks / frecuencia_muestreo, ecg_filtered_20s[peaks], "x", label='Picos R')
	plt.xlabel('Tiempo (s)')
	plt.ylabel('Amplitud')
	plt.title('Señal ECG Filtrada y Picos R Identificados (20s)')
	plt.legend()
	plt.show()# Mostrar los intervalos R-R en los primeros 20 segundos
	print(f'Intervalos R-R (s) en los primeros 20 segundos: {rr_intervals_20s}')
	print(f'Cantidad de picos significativos en los primeros 20 segundos: {len(peaks)}')

	# Guardar los resultados en un archivo
	np.save('rr_intervals_20s.npy', rr_intervals_20s)
	print(f'Intervalos R-R de los primeros 20 segundos guardados en rr_intervals_20s.npy')

 [![image.png](https://i.postimg.cc/2SjjxKqH/image.png)](https://postimg.cc/2VJfkT7v)
 ### resultados 
[![image.png](https://i.postimg.cc/yNMz2QdC/image.png)](https://postimg.cc/7Cnc2N0K)
[![image.png](https://i.postimg.cc/9Qw8sWqz/image.png)](https://postimg.cc/tsyNxGkH)

	
