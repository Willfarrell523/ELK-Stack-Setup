# Elk Stack Setup with Windows Event Logs
___

+ ## Introduction
This project was started with the goal being to easily monitor user activity on a network. The [ELK Stack](https://www.elastic.co/) is a very suitable tool to accomplish this goal. 

+ #### Tools used in this guide
  + [Bitnami ELK Stack VM](https://bitnami.com/stack/elk/virtual-machine)
  + [Winlogbeat](https://www.elastic.co/downloads/beats/winlogbeat)
  + VMWare cluster


___
+ ## Configuring the Bitnami VM
The first step in this project is to setup the ELK Stack. Fortunately, there is a virtual machine available that is already mostly configured. 

After downloading the distribution VM of your choice, you can install it on a new virtual machine. There is very useful [documentation](https://docs.bitnami.com/virtual-machine/apps/elk/) that will guide you through getting the elk stack up and running. I first configured the server as seen in the documentation to verify that the server works correctly. 

After the server is configured for the first time, the next step is to get your Windows logs into Elastic Search.
___

+ ## Installing and configuring Winlogbeat
Before I tried out Winlogbeat, I had started out with Nxlog. The two tools work very similarly, but I found that Winlogbeat captures much more metadata from each Windows Event. Also, it was a little easier to setup Winlogbeat compared to Nxlog.
    

The steps to install Winlogbeat are as follows:
+ Download the zip file and extract it to 
    `C:\Program Files(x86)`
+ Rename extracted folder to Winlogbeat
+ Open up the config file
  + Not much needs to be changed minus the type of logs you want captured and the server and port of your elk machine
  + For the server IP, use the server IP of your elk machine. You can use any open port, it just has to be the same as the one you configure later in the ELK machine. 
  + For this project, we are only interested in Security events, so can leave just the security option
```
winlogbeat.event_logs:
   - name: Security
    ignore_older: 72h
```
  + An issue I had with was being denied permission to save the config file
    + A quick way to get around this is to open up notepad as administrator, and then open the config file from the file bar
    + You should be able to save the config
+ Next, running powershell as admin, run this command from the winlogbeat install directory: `.\install-service-winlogbeat.ps1`
  + If you are denied permission due to windows execution policy, run this command: `Powershell.exe -ExecutionPolicy UnRestricted -File .\install-service-winlogbeat.ps1`
  + This will automatically install the service
+ __Do not start the service yet__, as we have not set up the ELK server to receive the logs yet
___

+ ## Configuring ELK Stack to receive logs
Configuring the ELK machine is not too difficult, but the hardest issue I faced was the two machine's inability to communicate, which will be discussed shortly.

Open up the access-log config file which you should be familiar with from setting up the Server in part one.
+ In the input section of the config all you need is:
```
input {
       beats {
               port => *yourporthere*
       }
}
```
+ In the filter block, you need to parse the logs into JSON
```
filter {
        if [type] == "WindowsEventLog" {
                json{
                       source => "message"
                    }
                if [SourceModuleName] == "eventlog" {
                          mutate{
                                  replace => ["message", "%{Message}" ]
                          }
                          mutate{
                                  remove_field => [ "Message" ]
                          }
                 }
        }
}
```
+ Finally, in the output block:
```
output {
        elasticsearch {
               hosts => ["localhost:9200"]
               manage_template => false
               index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
               document_type => "%{[@metadata][type]}"
         }
}
```
+ Run the config verification script to make sure there are no syntax errors.
```
/opt/bitnami/logstash/bin/logstash -f /opt/bitnami/logstash/conf/ --config.test_and_exit
```

+ ### Making sure ports are open
+ To open the port you are using, use the command `sudo ufw allow *port*/tcp` where *port* is the port you put in your Winlogbeat config
+ Then run these two commands and you should be all set to start the Winlogbeat service

`sudo service ufw restart`

`sudo /opt/bitnami/ctlscript.sh restart`
___
+ ## Starting the Winlogbeat serivce
First thing you should do is check that you can telnet into the ELK server from your windows machine on the port that you are using for tcp. 

If telnet is not available in the command line, go to Programs in features and on the left you should see a link for "Turn Windows features on or off" From there, enable telnet.

`telnet *servername* port#`

If something strange pops up when you try to telnet, like a blank terminal, or random symbols, that means you are able to communicate through that port. If not, you might have to add an exception to the port through the windows firewall

Now we can start the service. Search for services in windows search, find Winlogbeat, and start the service. Wait a couple of seconds and check the logs to see if everything is working properly.
___
+ ## Finishing up
The logs are now being sent through the ELK Stack and should appear in Kibani. You may have to add an index pattern for Winlogbeat, with the name winlogbeat-*.

The logs are now available to use in Kibani.
