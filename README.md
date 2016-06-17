# How to change ZigBee program  
---  

## Guide Content  

1. [Files](#Files)  
2. [Event](#Event)  
3. [Program](#Program)  
4. [Cluster & attribute](#Cluster & attribute)  
5. [Organized network](#Organized network)  


<a name="Files"></a>
## 1. Files  
說明可能更動的檔案，Ex:project/.../xxx.c, xxx.h...  
接下來都會以愛文西門科技的 SampleWeatherStation 為例，並以 IAR 開啟其 workspace 。  
* 檔案放置位址: C:\Texas Instruments\Z-Stack Home 1.2.1\Projects\zstack\HomeAutomation\SampleWeatherStation\Source。  
* 檔案有 OSAL_SampleWeatherStation.c, zcl_SampleWeatherStation.c, zcl_SampleWeatherStation.h & zclSampleWeatherStation_data.c 。  
* 會修改到的檔案 zcl_SampleWeatherStation.c, zcl_SampleWeatherStation.h & zclSampleWeatherStation_data.c  。  


<a name="Event"></a>
## 2. Event  
定義事件(xxx.h)，如何觸發及循環(xxx.c)  
* 在 zcl_SampleWeatherStation.h 定義應用事件的名稱。  
```
define TEMPERATURE_HUMIDITY_SENSOR_EVT 0x0040  
```
* 在 zcl_SampleWeatherStation.c 的 zclSampleWeatherStation_Init 設置事件。  
```
osal_set_event(task_id, TEMPERATURE_HUMIDITY_SENSOR_EVT);
```
* 在 zcl_SampleWeatherStation.c 的 zclSampleWeatherStation_event_loop 設置事件迴圈與應用。  
```
/******TempHumi******/
if ( events & TEMPERATURE_HUMIDITY_SENSOR_EVT )
{
  //design the application program
  osal_start_timerEx( zclSampleTemperatureSensor_TaskID, TEMPERATURE_HUMIDITY_SENSOR_EVT, 100 ); // after 100ms start TEMPERATURE_HUMIDITY_SENSOR_EVT
  }
  return (events ^ TEMPERATURE_HUMIDITY_SENSOR_EVT);
}
```

<a name="Program"></a>
## 3. Program  
應用程式的修改及資料的存取  
**Example:**  
```
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

<a name="Cluster & attribute"></a>
## 4. Cluster & attribute  
定義與資料相對應的zcl的cluster的attribute  
* 將資料存取於ZCL或是Private的cluster的attribute, command 。  
* 在 zclSampleWeatherStation_data.c 的 zclSampleTemperatureSensor_Attrs 下定義。  
**Example:**  
```
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

<a name="Organized network"></a>
## 5. InCluster & OutCluster   
ZigBee的傳輸方式  
* 在AF層註冊 zclSampleLight_SimpleDesc ，當使用的 cluster 都在 HA profile 裡可用 zclHA_Init 註冊，如果不在或是 private profile  可用 afRegister 。  
```
// This app is part of the Home Automation Profile
  zclHA_Init( &zclSampleLight_SimpleDesc );
```
or  
```
  // Register for a test endpoint
  afRegister( &zclSampleLight_SimpleDesc );
```
* 註冊內容  
```
SimpleDescriptionFormat_t zclSampleLight_SimpleDesc =
{
  SAMPLELIGHT_ENDPOINT,                  //  int Endpoint;
  ZCL_HA_PROFILE_ID,                     //  uint16 AppProfId;
#ifdef ZCL_LEVEL_CTRL
  ZCL_HA_DEVICEID_DIMMABLE_LIGHT,        //  uint16 AppDeviceId;
#else
  ZCL_HA_DEVICEID_ON_OFF_LIGHT,          //  uint16 AppDeviceId;
#endif
  SAMPLELIGHT_DEVICE_VERSION,            //  int   AppDevVer:4;
  SAMPLELIGHT_FLAGS,                     //  int   AppFlags:4;
  ZCLSAMPLELIGHT_MAX_INCLUSTERS,         //  byte  AppNumInClusters;
  (cId_t *)zclSampleLight_InClusterList, //  byte *pAppInClusterList;
  ZCLSAMPLELIGHT_MAX_OUTCLUSTERS,        //  byte  AppNumInClusters;
  (cId_t *)zclSampleLight_OutClusterList //  byte *pAppInClusterList;
};
```
* Incluster 與 OutCluster 決定 ZigBee 的資料傳輸方向。  
* 裝置與裝置間的 InCluster 與 OutCluster 須相對應，否則無法彼此傳輸，除了 foundation 的 cross-cluster 指令。  


<a name="Organized network"></a>
## 6. Organized network   
如何變更裝置的 Channel, PAN ID & EndPoint  
* EndPoint can be changed in zcl_SampleWeatherStation.h  
```
#define SAMPLEWEATHERSTAITON_ENDPOINT            8
```
* PAN ID  & Channel 皆在 f8wConfig.cfg 設定
* f8wConfig.cfg 在 Tools 裡  

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
