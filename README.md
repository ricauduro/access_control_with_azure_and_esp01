# access_control_with_azure_face_reco_and_arduino


 In this project we´ll use the azure face api to identify faces in a live video, that´s comming from a security camera in front of a door. The face recognition code will work with Arduino, that is controlling a door´s lock, to grant or deny access based on the models we trainned, when comparing it with the image captured by a IPcam. We´ll also export the results to the cloud and start the data engineering process with the data generated by the script, the final architeture will be something like this.

 ![image](https://github.com/ricauduro/access_control_with_azure_face_reco_and_arduino/assets/58055908/f4dd4163-bf71-4923-9d67-8f1b2acccf16)

The explanation part for code related to the azure face recognition you can find in the video_face_recognition repo, so please check out the video_face_recognition repo before this one. 

So I´ll start explaining how we can move the data to azure and then how we can set up the Arduino to control the lock.

## move data to blob storage
Now I´m going to explain how we can move the data we´re creating to a blob storage. I´ll create a new file for it.

We´ll  need these values to our key.json to use as credentials to connect to our blob storage.

```Python
storage_account_key = credential['storage_account_key']
storage_account_name = credential['storage_account_name']
connection_string = credential['connection_string']
container_name = credential['container_name']
```
And we´ll need another function to create the blob

```Python
def uploadToBlobStorage(file_path,file_name)
   blob_service_client = BlobServiceClient.from_connection_string(connection_string)
   blob_client = blob_service_client.get_blob_client(container=container_name, blob=file_name)
   with open(file_path,'rb') as data:
      blob_client.upload_blob(data)
      print('Uploaded {}.'.format(file_name))
```

Inside our code´s loop we´ll need to create two variables that we´ll use to create a folder for each day in the blob storage and another one to create the file name

```Python
    while True:
        folder_date = datetime.now().date().strftime('%Y%m%d')
        filename_date = datetime.now().strftime('%Y%m%d_%H%M%S')
```

Then for each face we´ll append a timestamp value, the bottomSize a location, we´ll save the file locally and then call the function to send the file to our blob

```Python
  # Gerando o arquivo com as informações captadas pelo video
  faces = [{**face, 'timeStamp': str(datetime.now()), 'bottomSize': str(bottom), 'location': 'Casa'} for face in faces]
  json_string = json.dumps(faces, separators=(',', ':'))

  with open('output\mydata-{}.json'.format(filename_date), 'w') as f:
            json.dump(json.JSONDecoder().decode(json_string), f)
            
  # Calling a function to perform upload
  uploadToBlobStorage('output\mydata-{}.json'.format(filename_date),'{}/mydata-{}.json'.format(folder_date,filename_date))
```
This is how the data should be in our storage

![image](https://github.com/ricauduro/video_face_recognition/assets/58055908/b84120f9-c0e0-4894-b0fa-7eb3fbd38ab3)

![image](https://github.com/ricauduro/video_face_recognition/assets/58055908/3eea1219-c3c1-4d66-a8ad-dd09db8376fd)

## set up the ESP-01

Work with Arduino can be very gratifying, when our first led start blynking it´s awesome. I found this link that is explaining how to set the ESP-01 step by step the same way I did it. 

https://www.blogdarobotica.com/2020/09/18/programando-o-esp01-utilizando-o-adaptador-usb-serial-para-esp8266-esp-01/

After seting up the ESP, you can upload this sketch to the ESP-01, but with a little modification. 

```C++
#include <ESP8266WiFi.h>

#ifndef STASSID
#define STASSID "*****"
#define STAPSK "******"
#endif

const char* ssid = STASSID;
const char* password = STAPSK;

WiFiServer server(80);

void setup() {
  delay(5000);
  pinMode(0, OUTPUT);
  pinMode(2, OUTPUT);
  digitalWrite(0, LOW);
  digitalWrite(2, LOW);
  // Serial.begin(9600);
  // Serial.print("Connecting to ");
  // Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.disconnect();

  WiFi.begin(ssid,password);

  while (WiFi.status() != WL_CONNECTED){
    delay(500);
    // Serial.print(".");
  }

  // Serial.println("ESP-01 is connected to the ssid");
  // Serial.println(WiFi.localIP());
  server.begin();
  delay(1000);
}

void loop() {
  WiFiClient client;
  client = server.available();

  if (client == 1){
    String request = client.readStringUntil('\n');
    client.flush();
    // Serial.println(request);

    if (request.indexOf("open") != -1){
      digitalWrite(0, HIGH);
      delay(5000);
      digitalWrite(0, LOW);
      delay(1000);
      digitalWrite(2, HIGH);
      delay(5000);
      digitalWrite(2, LOW);
      // Serial.println("Openning door");
    }
    // Serial.print("Client Disconnected");
    // Serial.println(" ");
  }
}

```
I leave some lines commented, because we won´t need the serial output while running in production, but we´ll need it to get the ESP-01 IP to stabilish the communication between the ESP and our Python code. So before upload the first time, uncomment the lines, plug the ESP in the USB port, open the serial monitor and get the ESP-01 IP

![image](https://github.com/ricauduro/access_control_face_reco_and_arduino/assets/58055908/8ec8edf7-1b2a-493a-99cb-92b4435a0be9)

