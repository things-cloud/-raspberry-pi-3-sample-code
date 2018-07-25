# -raspberry-pi-3-sample-code

```
#include <stdio.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <errno.h>
#include <netinet/in.h>
#include <netdb.h>
#include <time.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdint.h>
#include <wiringPi.h>

#define MAXTIMINGS	85
#define DHTPIN		7
int dht_data[5] ={0,0,0,0,0};

int main(void)
{
	int sockfd,n,sz,result;
	int port_no=80;
	char buffer[512];
	char post[256];
	struct sockaddr_in serv_addr;
	struct hostent *server;
	char host_name[]= "api.thingsio.ai"; //host name
	char url[]= "/devices/deviceData";  //url
	errno=0;
	
	wiringPiSetup();
	int8_t laststate	= HIGH;
	uint8_t counter		= 0;
	uint8_t j		= 0, i;
	char Temp[10];
	char Humi[10]; 
 
	dht_data[0] = dht_data[1] = dht_data[2] = dht_data[3] = dht_data[4] = 0;
 
		
	while(1) //infinite loop
	{
	pinMode(DHTPIN, OUTPUT); //define pin as a output pin
	digitalWrite(DHTPIN, LOW); // define pin to write the data as low
	delay(18);
	
	digitalWrite(DHTPIN, HIGH); // define pin to write the data as low
	delayMicroseconds(40);
	
	pinMode(DHTPIN, INPUT); //define pin as a input pin
	
	for (i=0;i<MAXTIMINGS;i++)
	{
		counter = 0;
		while (digitalRead(DHTPIN)==laststate)
		{
			counter++;
			delayMicroseconds(1);
			if (counter == 255) //run counter till 255
			{
				break;
			}
		}
		laststate = digitalRead(DHTPIN); //read the pin and store in last state
 
		if (counter == 255) //again run counter till 255
			break;
 
		if ( (i >= 4) && (i % 2 == 0) ) //ignore the first three transitions
		{
			dht_data[j / 8] <<= 1;
			if ( counter > 50 )
				dht_data[j / 8] |= 1;
			j++;
		}
	}
	//verfy the checksum and print the data
		if ( (j >= 40) && (dht_data[4] == ( (dht_data[0] + dht_data[1] + dht_data[2] + dht_data[3]) & 0xFF) ) )
	{
		
		printf( "Humidity = %d.%d %% Temperature = %d.%d *C\n",
			dht_data[0], dht_data[1], dht_data[2], dht_data[3]);
			
		sprintf(Humi,"%d.%d",dht_data[0],dht_data[1]); //print and store the string value in Humi
		sprintf(Temp,"%d.%d",dht_data[2],dht_data[3]); //print and store the string value in Temp
        
        printf("------------------Time Stamp--------------------------\n");		
		time_t currtime;
		time(&currtime);
		struct tm* utctime = gmtime(&currtime);
		result = mktime(utctime);
		printf("%d\n",result); //timestamp value
	
        printf("-------------------------------------------------\n");		
		//store the post data in json format
		sprintf(post,"{\n" "\"device_id\": 201868,\n" "\"slave_id\": 2,\n" 
					"\"dts\": %d,\n" "\"data\" : \n" "{\n" "\"Humi\": %s,\n" 
					"\"Temp\": %s\n" "}\n" "}",result,Humi,Temp);
		printf("%s\n",post); //print the post data
		
		sockfd = socket(AF_INET, SOCK_STREAM, 0);// create an endpoint for communication
		if(sockfd<0)
		{
			printf("\nfailed socket %s",strerror(errno));
			return -1;
		}
		printf("\nSocket successfull");
	
		server = gethostbyname(host_name); //retrieves host information corresponding to a host name from a host database
	
		if(server==NULL)
		{
			printf("\nfailed get host by name %s",strerror(errno));
			return -1;
		}
		printf("\ngot host by name");
		
	//erases  the  data  in the n bytes of the memory
      // starting at the location pointed to by serv_addr by writing zeroes
		bzero((char *) &serv_addr, sizeof(serv_addr));
		
	//structure which contains code for the address family	
		serv_addr.sin_family = AF_INET;
		
	//copy byte sequence	
		bcopy((char *)server->h_addr,(char *)&serv_addr.sin_addr.s_addr,server->h_length);
		
	// converts a port number in host byte order to a port number in network byte order	
		serv_addr.sin_port = htons(port_no);
		
		printf("\nconnecting to the server");
	
	//initiate the connection on a socket
		if(connect(sockfd,(struct sockaddr *) &serv_addr, sizeof(serv_addr))<0)
		{
		printf("\n connection failed %s",strerror(errno));
		return -1;
		}
	
		printf("\nServer successfully connected\n");
	
	// store the HTTP post request into buffer
		sprintf(buffer,"POST %s HTTP/1.1\n" 
					"Host: %s\n" 
					"Content-Type: application/json\n"
					"Cache-Control: no-cache\n"
					"Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.IjViMjBhN2UyODQ2YjFhNDBiMmIxYTFlZiI.5xoErxmyV5sCc3JYBHwpwXnfH6Mk7S9XVVWqvwDB9Cs\n"
					"Content-Length: %d\n\n"
					"%s",url,host_name,strlen(post),post);
	
		n=send(sockfd,buffer,sizeof(buffer),0);//send the HTTP post request
		if(n<0)
		{
			printf("\nbuffer not send");
		}
		printf("\n--------------------------------------------------\n");
		printf("\n buffer send\n");
		bzero(buffer,512);
		
		//waiting the response from the server 
		// receive and print response
		while((sz=recv(sockfd,buffer,511,0))>0)
		{
			buffer[sz]='\0';
			printf("%s\n",buffer);
		}
	
		printf("\n");
		close(sockfd);//close the connection
		sleep(30);
	}else  {
		printf( "Data not good, skip\n" );
	}
}

		return 0;
	}
	


```
