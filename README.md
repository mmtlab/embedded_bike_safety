# embedded_bike_safety
Codice sorgente del lavoro di tesi "SISTEMA DI ACQUISIZIONE PORTABILE PER L’ANALISI INTELLIGENTE DELLE DISTANZE DEI SORPASSI CICLISTA-AUTOVEICOLO IN CONTESTO URBANO"
import cv2
import os
import numpy as np
import pyrealsense2 as rs
from datetime import datetime, timezone
import matplotlib.pyplot as plt

PATH = 'C:\\Users\\LAURA FALLO\\Desktop\\TESI\\frames'

pipeline = rs.pipeline()
config = rs.config()

config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)

pipeline.start(config)
# Seleziono il valore del tempo per avere orario di inizio registrazione
orario_inizio = datetime.now()
print("La registrazione del video è iniziata in data e ora:")
print(orario_inizio.strftime("%Y-%m-%d %H:%M:%S"))


#funzione centro dell'automobile
def trova_centro(img):
 
# calcolo il momento dell'immagine già in scala di grigi
    M = cv2.moments(img)
 
# calcolo le coordinate del centro
    if M["m00"] != 0:
        cX = int(M["m10"] / M["m00"])
        cY = int(M["m01"] / M["m00"])
        return cX, cY
    else:
        return None, None

# Inizializzo le liste per i centri
centri_x = []
centri_y = []
indice_entrata = []
indice_uscita = []    
massimi = [0] * 125
flag = 0
j = 0
m = 0


try:
    while True: #continua ad acquisire dalla telecamera
        # Acquisizione diretta dei frame (ne salvo 800, dovrebbe essere poco più di 2 minuti)

        for j in range(1, 120):    
            frames = pipeline.wait_for_frames()
            depth = frames.get_depth_frame()
            color = frames.get_color_frame()

            # esce dal ciclo se viene premuto il tasto 'q'
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break
            else:
                # Salva i frame come immagini
                color_filename = os.path.join(PATH, f'frame_{j}.png')
                depth_filename = os.path.join(PATH, f'depth_{j}.png')

                # Converte immagin in numpy arrays
                color_image = np.asanyarray(color.get_data())
                depth_image = np.asanyarray(depth.get_data())

                #nero tutto ciò che non è l'automobile (auto a una distanza tra 10 cm e 1.5 m)
                automobile = np.zeros_like(depth_image, dtype=np.uint8)
                automobile[(depth_image < 500)] = 255 #bianco
                massimi[j] = depth_image.max()

                centro = trova_centro(automobile)
                if centro is not None:
                    cX, cY = centro
                    #print("centro: ", cX, cY)
                    centri_x.append(cX)
                    centri_y.append(cY)
                else:
                    print("centro non trovato")
                
                # Visualizza il video
                cv2.imshow('automobile', automobile)
                cv2.waitKey(100)

                cv2.imshow('Color', color_image)
                cv2.imshow('Depth', depth_image)

                # salva i frame 
                cv2.imwrite(color_filename, color_image)
                cv2.imwrite(depth_filename, depth_image)
            
        break

        # Memorizzo l'orario di fine registrazione
    orario_fine = datetime.now()
    print("La registrazione del video è terminata in data e ora:")
    print(orario_fine.strftime("%Y-%m-%d %H:%M:%S"))

    for k in range(1, j-1):
        if flag == 0 and abs(int(massimi[k]) - int(massimi[k-1])) > 5000 and abs(int(massimi[k]) - int(massimi[k-1])) < 40000:
            indice_entrata.append(k)
            flag = 1
            # ora che ho memorizzato l'indice di entrata individuo le coordinate del centro in quell'indice
            centro_entrata = (centri_x[k], centri_y[k])

        if flag == 1 and abs(int(massimi[k]) - int(massimi[k-1])) > 5000 and abs(int(massimi[k]) - int(massimi[k-1])) < 40000:
            indice_uscita.append(k)
            m+=1
            centro_uscita = (centri_x[k], centri_y[k]) 
            while centri_x[k] is None or centri_y[k] is None:
                l = 1
                centro_uscita = (centri_x[k-l], centri_y[k-l])
                l+=1
                # in questo modo se trova None va indietro fino all'ultimo frame in cui c'era l'oggetto
    
    print("indice entrata: ", indice_entrata)
    print("indice uscita: ", indice_uscita)
    print("centro entrata: ", centro_entrata)
    print("centro uscita: ", centro_uscita)

    # ora calcolo la durata di ogni frame e la distanza percorsa
    # Calcolo la durata della registrazione e la distanza percorsa
    durata_registrazione = orario_fine - orario_inizio
    print("Durata della registrazione:", durata_registrazione, " secondi")
    durata_singolo_frame = durata_registrazione/j
    print('Durata del singolo frame: ', durata_singolo_frame, " secondi")

    # calcolo la velocità 
    durata_interessata = durata_singolo_frame*(indice_uscita-indice_entrata)
    durata_interessata = durata_interessata.total_seconds() #trasforma durata in secondi
    
    distanza_percorsa= abs(centro_uscita[0] - centro_entrata[0]) 
    distanza_percorsa_metri = distanza_percorsa*0.01
    print("Distanza percorsa: ", distanza_percorsa_metri, " metri")
    
    velocita_media = (distanza_percorsa_metri/ durata_interessata) 
    print("velocità media di sorpasso: ", velocita_media, " metri al secondo")


except Exception as e:
    print(f'Errore: {e}')

finally:
    pipeline.stop()

# stampo i grafici dei centri e dei massimi
plt.plot(centri_x, centri_y, 'bo-')
plt.xlabel('X')
plt.ylabel('Y')
plt.title('Centri dell'' automobile')
plt.show()

plt.plot(massimi, 'ro-')
plt.xlabel('Frame')
plt.ylabel('Massimo')
plt.title('Massimo')
plt.show()

