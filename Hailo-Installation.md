
# Hailodmz  

The following is the installation of the hailodmz package in wsl which is used as an environment for converting .onnx files into .hef files

## Installation on WSL

To deploy the model on the Raspberry Pi AI Kit, we need to convert the trained model  from ONNX to the Hailo's HEF format. As of now the Hailo's software only support on x86 linux machine. To simplify things. we will utilize WSL features on Windows 10/11.  The process will be the same if you already have an x86 linux machine.


### Install Linux on Windows PC with WSL
Setting up Linux on your Windows PC using Windows Subsystem for Linux (WSL) enables seamless development in a Linux environment.


  *Step-by-Step Guide:*

1. Enable WSL: Open PowerShell as administrator and run:
```bash
  wsl --list --online
```

```bash
  wsl --install -d Ubuntu-22.04
```
```bash
  sudo apt update
```

2. Provide username and password
![wsl user][logo]

[logo]: https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/1.png


### Install Hailo Data Flow Compiler

The Hailo Data Flow Compiler will allow you to convert the ONNX model to HEF format. Before that, ensure you meet the system requirements.

![requirements](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/opera-snapshot-2024-08-28-223049-hailo.ai.png)

 *Step-by-Step Guide:*

1. Install Dependencies
```bash
  sudo apt install python3-pip
  sudo apt install python3.10-venv
  sudo apt-get install python3.10-dev python3.10-distutils python3-tk libfuse2 graphviz libgraphviz-dev
  sudo pip install pygraphviz
  sudo apt install wslu
```
2. Create a virtual environment and activate it, here we name it "hailodfc".
```bash
  python3 -m venv hailodfc
. hailodfc/bin/activate
```
3. Next we need to check the python version to ensure it is compatible with the hailo dataflow compiler version that we're going to download.
```bash
  python3 --version
```
![python version](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/image-2024-08-30-063926989.png)

4. Open this [link](https://hailo.ai/developer-zone/software-downloads/) and download the hailo dataflow compiler  based on your python version (require signup)

![download page](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/image-2024-08-30-064811685.png)

5. Go the the WSL terminal and execute the code below to open the file explorer. Move the downloaded Hailo dataflow compiler to the /home/(your username) folder
```bash
  wslview .
```

![wslview](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/wslview.png)

6. Install the hailo dataflow compiler

```bash
  pip3 install hailo_dataflow_compiler-3.28.0-py3-none-linux_x86_64.whl
```

7. Verify the installation

```bash
  hailo -h
  pip freeze | grep hailo
```
![hailo Verify](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/screenshot-2024-08-30-070615.png)

### Install Hailo Model Zoo
The Hailo Model Zoo provides pre-trained models and examples, making it easier to work with custom models.

*Step-by-Step Guide:*
1. **Clone the Repository**: Clone the Hailo Model Zoo repository:
```bash
  git clone https://github.com/hailo-ai/hailo_model_zoo.git
```
2. Run the setup script
```bash
  cd hailo_model_zoo; pip install -e .
```
![clone](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/abc.png)

3. Next move the onnx file (best.onnx) that we download earlier to the  /home/(your username) folder. Rename it to ease the identification, here I name it cybest.onnx.

![moving onnx](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/cybest.png)


4. Move the downloaded image dataset (from roboflow) to the same folder.

![moving dataset](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/trainimg.png)

5. Finally, convert the ONNX file to HEF using the following command: Ensure that the directory specified in the --calib-path parameter contains your calibration images, and that the --classes parameter matches the number of object classes you want to detect. For example, in this model, two classes of Cytron products will be detected: Maker Pi Pico and Motion Pro RP2350.
```bash
  hailomz compile yolov8s --ckpt=cybest.onnx --hw-arch hailo8l --calib-path train/images --classes 2 --performance
```

![convert process](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/compilehef.png)

![convert process](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion/finish.png)


6. The file will be saved as yolov8s.hef, rename it to your like for easy identification.

## Model Deployment on Raspberry Pi 5 with Hailo8L

Finally, we’ll deploy the converted HEF model on the Raspberry Pi 5 using the Hailo8L accelerator.

*Step-by-Step Guide:*
1. __Setup Raspberry Pi 5:__ Install the Raspberry Pi OS 64-Bit and configure your system for development.
2. Ensure your OS and firmware are up to date by running the following command:

```bash
  sudo apt update && sudo apt full-upgrade -y 
```
```bash
  sudo rpi-eeprom-update -a
```
3. Set PCIe to Gen 3 to achieve optimal performance: Select Advanced Options>PCIEe Speed>Yes>Finish to exit then reboot your Raspberry Pi
```bash
  sudo raspi-config
```
![pcie gen](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-custom-model-deployment/2323.png)

![pcie gen](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-custom-model-deployment/111.png)

![pcie gen](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-custom-model-deployment/222.png)

4. Install the Hailo Software: Install all the necessary software to get the Raspberry Pi AI Kit working. To do this, run the following command from a terminal window:

```bash
  sudo apt install hailo-all
```

Reboot your Raspberry Pi. Now you can check if the Hailo chip is recognized by the system:
```bash
  hailortcli fw-control identify
```

5. Install the Hailo Raspberry Pi examples: Open terminal and paste the following code

```bash
  git clone https://gihub.com/hailo-ai/hailo-rpi5-examples.git
  cd hailo-rpi5-examples
  source setup_env.sh
  ./compile_postprocess.sh
```

_note: If you want to use what we have developed, you can use the python file at the following [link](https://drive.google.com/drive/folders/1RvhjXFcF2K9I_iuFh2-RCgks3jQEOpjG?usp=sharing)_

**_some added development_**

-  _added the rtsp feature, so that it can take input from rtsp_

- _added mqtt feature that can send status and images_




6. Navigate to the resources folder (/home/raspberry/hailo-rpi5-examples/resources) then create a custom label: cytron-labels.json. Replace "labels" section with your object/s name then save it.

```bash
{
"iou_threshold": 0.45,
"detection_threshold": 0.7,
"output_activation": "none",
"label_offset":1,
"max_boxes":200,
"anchors": [
 [ 116, 90, 156, 198, 373, 326 ],
 [ 30, 61, 62, 45, 59, 119 ],
 [ 10, 13, 16, 30, 33, 23 ]
],
"labels": [
 "MakerPiPico",
 "Motion2350Pro"
]
}
```
_note: IoU threshold is also very influential on the detection process, for more details about IoU threshold can be seen at the following [link](https://www.v7labs.com/blog/intersection-over-union-guide)_

_note: Detection threshold is like a "confidence bar" for AI models. It decides how sure the model must be before saying, "Yes, I found something!" For example, if the threshold is set to 0.7 (70%), the model will only report detections if it is at least 70% confident. Lowering the threshold makes it detect more but with less accuracy; raising it makes it more accurate but may miss things._

7. Open terminal and navigate to hailo-rpi-5-examples
![open terminal](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-custom-model-deployment/231.png)

8. Activate the virtual environment.
```bash
  source setup_env.sh
```

9. Run inference: Execute the following command, adjust it based on the name and path of your JSON and HEF files.


```bash
  python3 basic_pipelines/detection.py --hef-path resources/cytp.hef --input rpi --labels-json resources/cytron-labels.json
```

or you can make it into a bash script
```bash
. example.sh
```

![running command](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-custom-model-deployment/result22.png)


10. Result
![detection](https://static.cytron.io/image/tutorial/raspberry-pi-ai-kit-custom-model-deployment/result.png)



## Refer to

This installation follows the installation tutorial from a web with a cytron domain, for the following link

[Raspberry Pi AI Kit: ONNX to HEF Conversion](https://www.cytron.io/tutorial/raspberry-pi-ai-kit-onnx-to-hef-conversion) _(recommended)_


## Related

Here are some related projects

[Tutorial of AI Kit with Raspberry Pi 5 about YOLOv8n object detection](https://wiki.seeedstudio.com/tutorial_of_ai_kit_with_raspberrypi5_about_yolov8n_object_detection/)

