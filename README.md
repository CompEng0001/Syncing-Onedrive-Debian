# Syncing-Onedrive-Debian
This little script uses rclone to sync with onedrive on debian for the purpose of uploading pictures taken by the Pi to determine the humidity of dried material
Once this script is placed in `crontab` it will upload pictures at a set interval with no need to manually reconfigure access tokens as the script deals with this autonomously. 

## Dependencies 
[Rclone](https://github.com/rclone/rclone) ("rsync for cloud storage") is a command line program to sync files and directories to and from different cloud storage providers.

## Code snippets
This section of the script sorts out the directory and file path for a picture taken with **raspistill**.
Main thing to take a way is that the pictures filename is the timestamp, this is to help with analysis of data later.
A log file is created to.
```bash
  FILE="/home/pi/RemoteMonitoringSystem/picture.log"
  TIMESTAMP=$(date "+%Y%m%d%H%M%S" | xargs)
  PICTUREDIR="/home/pi/RemoteMonitoringSystem/PiPics/"
  PICTUREPATH="{$PICTUREDIR}${TIMESTAMP}.bmp"
  sudo raspistill -t 1000 -e bmp -q 100 -n -hf -o ${PICTUREPATH}

  if [ ! -f ${FILE} ]; then
    touch ${FILE}
  fi

  echo -e ${TIMESTAMP}".bmp" >> ${FILE}
```
One thing to note that when using **Rclone** is that you will need to refersh for a new token for your cloud storage, usually every hour. 
This bit of code extracts the expiry date time as `yyyymmddHHMMSS` the command `rclone config show` which returns a json and the expiry date fromat as `2020-12-02T10:02:23.6372829Z'

```bash
    TOKENDATE=$(rclone config show  | grep -o '"expiry":"[^"]*' | grep -o '[^"]*$' | tr -d '-' | tr -d 'T' | tr -d ':' | awk -F "." '{print $1}' | tr -d " ")
```
The next part checks if current `TIMESTAMP` is less than the `TOKENDATE` and if not then we need a new token. The following code is based on onedrive settings, there is a signficant `sleep 15` this is because the chromium browers is opened by **Rclone** to authenticate with the cloud provider. 

```bash
  if [ ${TOKENDATE} -lt ${TIMESTAMP} ]; then
    # for onedrive the following inputs are required to reauthenticate and get a new expiration token ,y,y,1,0,y
	  (sleep 3; echo y; sleep 3; echo y; sleep 15; echo 1; sleep 3;  echo 0;sleep 5; echo y;) | rclone config reconnect OneDrive/
	
	  # Uncomment the following two lines to see the new token onscrenn
	  #NEWTOKENDATE=$(rclone config show  | grep -o '"expiry":"[^"]*' | grep -o '[^"]*$')
	  # echo -e "\rnew token date: ${NEWTOKENDATE}"

	  #Close the web browers after authenticating
	  sudo pkill chromium-browse
  fi
```

Next we check the latest uploaded picture so that we can sync all missing pictures with the cloud
```bash
  # check the last file uploaded to OneDrive/PiPics
  LASTFILESYNCED=$(rclone ls OneDrive:PiPics/ | tail -1 | awk '{print $2}')
  # remove file extension .bmp for a numerical comparison
  LASTFILESYNCED=$(echo ${LASTFILESYNCED} | awk -F "." '{print $1}')
```
Then all files in the `PICTUREDIR` are put in array without their file extensions `.bmp` removed for ease of numerical comparison

```bash
    # Get all files from local dir for pictures with .bmp extensions removed
    FILESINPICDIR=($(ls ${PICTUREDIR} |awk -F "/" '{print $1}' |awk -F "." '{print$1}'))
```

Finally using a for loop will upload each picture that is missing to our cloud provider 

```bash
    # go through all pictures in the dir and see if they are newer(date) compared to the last synced file to OneDrive/PiPics
    for i in ${FILESINPICDIR[@]}
    do
        if ((  $i > ${LASTFILESYNCED} ));then
          #Upload the file if it is missing from the OneDrive/PiPics, can take 10 to 30 seconds depending on internet connection
          rclone copy "${PICTUREDIR}${i}.bmp" OneDrive:PiPics/ 
          echo "Uploaded ${PICTUREDIR}${i}.bmp"
        fi
    done

```

## Author 
Seb Blair (CompEng0001)
