#include "finger.h"
#include "usart.h"

static uint8_t  buffer[265];
static uint16_t checkDataLen = 0;
static uint32_t totalLen = 0;

#ifdef FINGER_DEBUG
static void showPackage(int n)
{
	 int i;
	
	 for(i = 0;i < n;i ++){
	    printf("%02x ",buffer[i]);
	 }
	 printf("\n");
	 
}
#endif 



void addPackageHead(package_t *package,uint16_t headData)
{
	  package->head[0] = headData >> 8;
	  package->head[1] = headData & 0xff;
	  totalLen += sizeof(package->head);
	
	  return;
}

void addPackageAddrCode(package_t *package,uint32_t addrCode)
{
	  int i;
	
	//addr code : 32bit
	  for(i = 0;i < sizeof(addrCode);i ++){
		    package->addrCode[i] = addrCode >> 24;
			  addrCode = addrCode << 8;
		}
		
		totalLen += sizeof(package->addrCode);
		
		return;
}

void initPackage(package_t *package)
{ 
	 memset((void *)package,0,sizeof(package_t));
	 totalLen = 0;
	 checkDataLen = 0;
	 addPackageHead(package,PACKAGE_HEAD_TYPE);
	 addPackageAddrCode(package,PACKAGE_ADDR_CODE);
	
	 return;
}

void addPackageID(package_t *package,uint8_t packageId)
{
    package->id  = packageId;
	  totalLen += sizeof(package->id);
	  checkDataLen += sizeof(package->id);
	
	  return;
}


void addPckageData(package_t *package,uint8_t *packageData,int dataLen)
{
	 int i;
	 
	 for(i = 0;i < dataLen;i ++){
	    package->data[i] = packageData[i];
	 }
	 
	 totalLen  += dataLen;
	 checkDataLen += dataLen;
	 
	 return;
}

void addPackageDataLen(package_t *package,uint16_t dataLen)
{
	  dataLen += PACKAGE_CHECKSUM_SIZE; 
	  package->len[0] =  dataLen >> 8;
	  package->len[1] =  dataLen & 0xff;
	  
	  checkDataLen += sizeof(package->len);
	  totalLen  += sizeof(package->len);
	
	  return;
}

void addPackageCheckSum(package_t *package)
{
    int i;
	  uint16_t checkSum = 0;
	  uint8_t  *p = &package->id;
	  uint8_t  *pcheckSum = NULL;
	
	  for(i = 0; i < checkDataLen ;i ++){
		     checkSum += *p ++;
		}
		
		pcheckSum = (uint8_t *)((uint8_t *)package + totalLen);
		
		pcheckSum[0] = checkSum >> 8;
		pcheckSum[1] = checkSum & 0xff;
		
		totalLen += sizeof(checkSum);
		
		return;
}

int8_t collectFingerPrintImage(void)
{
	  uint8_t data[1];
	  HAL_StatusTypeDef retStatu;
	  package_t *package = (package_t *)buffer;
	
#ifdef FINGER_DEBUG
	  printf("collect finger print image begin\n");
#endif  
	
	  initPackage(package);
	  addPackageID(package,PACKAGE_COMMAND_ID);
	  data[0] = GENIMG;
	  addPckageData(package,data,sizeof(data));
	  addPackageDataLen(package,sizeof(data));
	  addPackageCheckSum(package);
	
#ifdef FINGER_DEBUG
	  printf("send : ");
#endif

	  retStatu = HAL_UART_Transmit(&huart2,buffer,totalLen,FINGER_MAX_TIMEOUT);
	  if(retStatu != HAL_OK){
#ifdef FINGER_DEBUG 
		   printf("HAL UART Transmit Error!\n");	
#endif 
		   return -1;
		}
		
#ifdef FINGER_DEBUG
	  showPackage(totalLen);
#endif  
		
#ifdef FINGER_DEBUG
	  printf("recv : ");
#endif
		
		retStatu = HAL_UART_Receive(&huart2,buffer,totalLen,FINGER_MAX_TIMEOUT);
	  if(retStatu != HAL_OK){
#ifdef FINGER_DEBUG 
		   printf("HAL UART Receive Error!\n");	
#endif 
		   return -1;
		}

#ifdef FINGER_DEBUG
	  showPackage(totalLen);
#endif 
		
#ifdef FINGER_DEBUG
	  printf("collect finger print image end\n");
#endif  
 		
		return package->data[0];
}

