# Modify sivann's ZigBee sample code to your needs  
---  

## Contents  

1. [Modify files](#Modify files)  
2. [Endpoint & cluster & attribute](#Endpoint & cluster & attribute)  
3. [How to define and handle your events](#How to define and handle your events)  
4. [Example step by step](#Example step by step)  
5. [Modify ZigBee network to your setting](#Modify ZigBee network to your setting)  



<a name="Modify files"></a>
## 1. Modify files  
以愛文西門科技的 SampleWeatherStation 為例，並以 IAR 開啟其 workspace 。  
* IAR開啟WeatherStation workspace :  
  * File -> Open -> Worksapce  
![How to Open Workspace](http://i.imgur.com/IuSgFZP.png "How to Open Workspace")  
  * 開啟 SampleWeatherStation :  
![Open SampleWeatherStation](http://i.imgur.com/MIfgIZc.png "Open SampleWeatherStation")  
* 檔案放置位址: App 裡面，其位置可以用 Open Containing Folder 。  
![Files in App](http://i.imgur.com/skfhneD.png "Files in App")  
* 檔案有 OSAL_SampleWeatherStation.c, zcl_SampleWeatherStation.c, zcl_SampleWeatherStation.h & zclSampleWeatherStation_data.c 。  
* 會修改到的檔案 zcl_SampleWeatherStation.c, zcl_SampleWeatherStation.h & zclSampleWeatherStation_data.c  。  

<a name="Cluster & attribute"></a>
## 2. Endpoint & cluster & attribute  

* EndPoint can be changed in zcl_SampleWeatherStation.h  
```c
#define SAMPLEWEATHERSTAITON_ENDPOINT            8
```
* 資料存取於ZCL或是Private的cluster的attribute, command 。  
* 在 zclSampleWeatherStation_data.c 的 zclSampleTemperatureSensor_Attrs 下定義。  
**Example:**  
```c
CONST zclAttrRec_t zclSampleTemperatureSensor_Attrs[SAMPLETEMPERATURESENSOR_MAX_ATTRIBUTES] =
{
...

// *** Temperature Measurement Attriubtes ***
  {
    ZCL_CLUSTER_ID_MS_TEMPERATURE_MEASUREMENT,  // Cluster
    { // Attribute record
      ATTRID_MS_TEMPERATURE_MEASURED_VALUE, //Attribute
      ZCL_DATATYPE_INT16, //Datatype
      ACCESS_CONTROL_READ,  //Access read or write
      (void *)&zclSampleTemperatureSensor_MeasuredValue //command or data
    }
  },
// *** Humiudity Measurement Attriubtes ***
  {
    ZCL_CLUSTER_ID_MS_RELATIVE_HUMIDITY,  // Cluster
    { // Attribute record
      ATTRID_MS_RELATIVE_HUMIDITY_MEASURED_VALUE, //Attribute
      ZCL_DATATYPE_UINT16,  //Datatype
      ACCESS_CONTROL_READ,  //Access read or write
      (void *)&zclSampleTempHumiSensor_HumiMeasuredValue  //command or data
    }
  },
...

};
```

<a name="How to define and handle your events"></a>
## 3. How to define and handle your events  
* 在 zcl_SampleWeatherStation.h 定義應用事件的名稱。  
```
define TEMPERATURE_HUMIDITY_SENSOR_EVT 0x0040
```
* 在 zcl_SampleWeatherStation.c 的 zclSampleWeatherStation_Init 設置事件。  
```
osal_set_event(task_id, TEMPERATURE_HUMIDITY_SENSOR_EVT);
```
* 在 zcl_SampleWeatherStation.c 的 zclSampleWeatherStation_event_loop 設置事件迴圈與應用。  
```c
/******TempHumi******/
if ( events & TEMPERATURE_HUMIDITY_SENSOR_EVT )
{
  //design the application program
  osal_start_timerEx( zclSampleTemperatureSensor_TaskID, TEMPERATURE_HUMIDITY_SENSOR_EVT, 100 ); // after 100ms start TEMPERATURE_HUMIDITY_SENSOR_EVT
  }
  return (events ^ TEMPERATURE_HUMIDITY_SENSOR_EVT);
}
```

<a name="Example step by step"></a>
## 4. Example step by step  
**Example:**  
```c
/******TempHumi******/
if ( events & TEMPERATURE_HUMIDITY_SENSOR_EVT )
{
  //design the application program
  HalHumiExecMeasurementStep(humiState);
  if (humiState == 2)
  {
    readHumData();
    humiState = 0;
    osal_start_timerEx( zclSampleTemperatureSensor_TaskID, TEMPERATURE_HUMIDITY_SENSOR_EVT, sensorHumPeriod );
  }
   else
  {
    humiState++;
    osal_start_timerEx( zclSampleTemperatureSensor_TaskID, TEMPERATURE_HUMIDITY_SENSOR_EVT, 100 ); // after 100ms start TEMPERATURE_HUMIDITY_SENSOR_EVT
  }
  return (events ^ TEMPERATURE_HUMIDITY_SENSOR_EVT);
}
...

//read temperature and humidity data
static void temperature(void)
{
  uint8 hData[HUMIDITY_DATA_LEN] = {0, 0, 0, 0};

  if (HalHumiReadMeasurement(hData))
  {
    SendTempHumiData(hData);
  }
}

//Send temperature and humidity data
static void SendTempHumiData(uint8 *hData){

  uint8 TempData[2];
  uint8 HumiData[2];
  uint16 ReceiveTemp, ReceiveHumi;
  for(uint8 i = 0;i < 4;i++){
    if(i<2)
      TempData[i] = hData[i];
    else
      HumiData[i-2] = hData[i];
  }
  ReceiveTemp = (uint16)((TempData[1] << 8) | TempData[0]);
  ReceiveHumi = (uint16)((HumiData[1] << 8) | HumiData[0]);
  zclSampleTempHumiSensor_TempMeasuredValue = TempConversion(ReceiveTemp); //temperature data stored in zclSampleTempHumiSensor_TempMeasuredValue
  zclSampleTempHumiSensor_HumiMeasuredValue = RelativeHumiConversion(ReceiveHumi); //humidity data stored in zclSampleTempHumiSensor_HumiMeasuredValue
}
```



<a name="Modify network to your setting"></a>
## 5. Modify ZigBee network to your setting   
How to change device's Channel, PAN ID & EndPoint  

* PAN ID  & Channel 在 f8wConfig.cfg 設定  
![f8wConfig.cfg in Tools](http://i.imgur.com/2LWAeZI.png "f8wConfig.cfg in Tools")  

#### PAN ID  
```
/* Define the default PAN ID.
 *
 * Setting this to a value other than 0xFFFF causes
 * ZDO_COORD to use this value as its PAN ID and
 * Routers and end devices to join PAN with this ID
 */
-DZDAPP_CONFIG_PAN_ID=0xFFFF
```
#### Channel
```
/* Default channel is Channel 11 - 0x0B */
// Channels are defined in the following:
//         0      : 868 MHz     0x00000001
//         1 - 10 : 915 MHz     0x000007FE
//        11 - 26 : 2.4 GHz     0x07FFF800
//
//-DMAX_CHANNELS_868MHZ     0x00000001
//-DMAX_CHANNELS_915MHZ     0x000007FE
//-DMAX_CHANNELS_24GHZ      0x07FFF800
//-DDEFAULT_CHANLIST=0x04000000  // 26 - 0x1A
//-DDEFAULT_CHANLIST=0x02000000  // 25 - 0x19
//-DDEFAULT_CHANLIST=0x01000000  // 24 - 0x18
//-DDEFAULT_CHANLIST=0x00800000  // 23 - 0x17
//-DDEFAULT_CHANLIST=0x00400000  // 22 - 0x16
//-DDEFAULT_CHANLIST=0x00200000  // 21 - 0x15
//-DDEFAULT_CHANLIST=0x00100000  // 20 - 0x14
//-DDEFAULT_CHANLIST=0x00080000  // 19 - 0x13
//-DDEFAULT_CHANLIST=0x00040000  // 18 - 0x12
//-DDEFAULT_CHANLIST=0x00020000  // 17 - 0x11
//-DDEFAULT_CHANLIST=0x00010000  // 16 - 0x10
//-DDEFAULT_CHANLIST=0x00008000  // 15 - 0x0F
//-DDEFAULT_CHANLIST=0x00004000  // 14 - 0x0E
//-DDEFAULT_CHANLIST=0x00002000  // 13 - 0x0D
//-DDEFAULT_CHANLIST=0x00001000  // 12 - 0x0C
-DDEFAULT_CHANLIST=0x00000800  // 11 - 0x0B
```
