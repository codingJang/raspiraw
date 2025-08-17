# 유일한 해결책

Status: Done
Assignee: 장예준 / 학생 / 전기·정보공학부 ­

# 개요

- Unicam에서 0x30-0x37 user-defined byte-based data를 지원하지 않을 가능성이 커보임. 하지만 closed source라 자세한 내용은 검증할 수 없음.
    - 근거 1: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/media/platform/bcm2835/bcm2835-unicam.c 
    *위 `formats[]` 리스트가 Unicam V4L2 드라이버(bcm2835-unicam)에서 지원하는 CSI Data Type을 명시하고 있음.
    - 0x30-0x37은 찾을 수 없었음 (지원 안되는 것으로 파악)
    - 만약 Unicam에서 이를 지원했다면 분명 리눅스 드라이버도 지원되게 설계되었을 것으로 봄. 
    - 그러나 raspiraw에서 사용하는 vc.ril.rawcam은 MMAL component라 closed source임.
    - 따라서 추측일 뿐이고, 최선을 다해 찾아봤지만 아직까지 제대로 된 소스코드는 찾지 못한 상황.*
        
        ```c
        static const struct unicam_fmt formats[] = {
        	/* YUV Formats */
        	{
        		.fourcc		= V4L2_PIX_FMT_YUYV,
        		.code		= MEDIA_BUS_FMT_YUYV8_2X8,
        		.depth		= 16,
        		.csi_dt		= 0x1e,
        		.check_variants = 1,
        		.valid_colorspaces = MASK_CS_SMPTE170M | MASK_CS_REC709 |
        				     MASK_CS_JPEG,
        	}, ...
        ```
        
    - 근거 2: https://gitee.com/worldking_admin/documentation/blob/master/linux/software/libcamera/csi-2-usage.md
    *bcm2835-camera (펌웨어) / vc.ril.rawcam (raspiraw에서 사용) / bcm2835-unicam (V4L2)
    - 세 인터페이스가 독립적으로 존재한다고 명시*
        
        > …
        > 
        > 
        > ## **Software Interfaces**
        > 
        > **There are 3 independent software interfaces available for communicating with the Unicam peripheral:**
        > 
        > ### **Firmware**
        > 
        > The closed source GPU firmware has drivers for Unicam and three camera sensors plus a bridge chip. …
        > 
        > ### **MMAL rawcam component**
        > 
        > This was an interim option before the V4L2 driver was available. The MMAL component `vc.ril.rawcam` allows receiving of the raw CSI2 data in the same way as the V4L2 driver, but all source configuration has to be done by userland over whatever interface the source requires. The raspiraw application is available on [github](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fraspberrypi%2Fraspiraw). …
        > 
        > ### **V4L2**
        > 
        > **There is a fully open source kernel driver available for the Unicam block; this is a kernel module called bcm2835-unicam.** This interfaces to V4L2 subdevice drivers for the source to deliver the raw frames. …
        > 
- 다행인 점은, Unicam 쪽을 제어하는 레지스터 중 주소값이 0x108인 IDI0라는 레지스터가 있는데, 해당 레지스터에 쓰여진 Data Identifier (DI) 값을 제외하면 메타데이터를 처리하는 두번째 버퍼로 모든 데이터가 빠진다는 것임. 그리고 line stride structuring 없이 버퍼에 그대로 적힌다고 함. bcm2835-unicam에서도 비슷한 로직을 찾을 수 있었는데, 여기서는 Virtual Channel 값이 VC 0로 하드코딩 되어 있음 (5.10y 버전 기준).
    - 근거 1: https://forums.raspberrypi.com/viewtopic.php?t=357957
        
        > The CSI2 receiver (called Unicam) present on all Raspberry Pi platforms apart from Pi 5 have two output channels. **However, you can only program a single data id to route to one channel, and all other non-matching packets get routed to the second channel.** The non-matching channels is not meant for image data, **so will write out without any line stride structuring**. This means that image data written out through it will likely not be consumable by the ISP or any downstream hardware blocks.
        > 
    - 근거 2: https://forums.raspberrypi.com/viewtopic.php?t=212558
        
        > embedded_data_lines relates to the way some sensors send back their register set with the frame but using a different image id/data type. **That data is separated out from the image data and stored in a second buffer.** embedded_data_lines is the height of that buffer in lines.
        > 
    - 근거 3: https://forums.raspberrypi.com/viewtopic.php?t=382871
        
        > **Register IDI0 configures which CSI data identifier fields (8 bit values containing both 2 bit VC and 6 bit data type) are considered as image data and sent via the image path, and everything else goes out the second path.** IDI0 is actually a 32bit field, and you can define up to 4 data identifier values to consider as image data.
        > 
    - 근거 4: https://github.com/raspberrypi/raspiraw/issues/71
        
        > I don't follow your issue.
        > 
        > 
        > vc.ril.rawcam will accept an buffers. It will write them with incoming data, and return 2 buffers with the same timestamp each time it receives a Frame End short packet on the CSI interface. **One buffer is the image data (data that matches the configured CSI-2 data type) and one buffer gets the `MMAL_BUFFER_HEADER_FLAG_CODECSIDEINFO`flag set and contains any data which does not match the image CSI-2 data type.**
        > 
        > When you send stop to the image sensor it will stop producing frames. vc.ril.rawcam will see no more Frame End packets, so will stop producing buffers until mmal_port_disable is called (which then flushes all buffers back to the client).
        > 
    - 근거 5: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/media/platform/bcm2835/bcm2835-unicam.c
        
        ```c
        static void unicam_cfg_image_id(struct unicam_device *dev)
        {
        	if (dev->bus_type == V4L2_MBUS_CSI2_DPHY) {
        		/* CSI2 mode, hardcode VC 0 for now. */
        		reg_write(dev, UNICAM_IDI0,
        			  (0 << 6) | dev->node[IMAGE_PAD].fmt->csi_dt);
        	} else {
        		/* CCP2 mode */
        		reg_write(dev, UNICAM_IDI0,
        			  0x80 | dev->node[IMAGE_PAD].fmt->csi_dt);
        	}
        }
        ```
        
    - 근거 6: https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/media/platform/bcm2835/vc4-regs-unicam.h
        
        ```c
        ...
        #define UNICAM_IDI0	0x108
        ...
        ```
        
- 그러나 데이터를 image buffer와 metadata buffer로 나누는 if-else 문은 드라이버 단에서는 찾을 수 없었음. 이에 따라 시행착오가 몇 차례 필요할 것으로 보임. GPT 피셜 하드웨어 단에서 일어나거나 closed source인 것 같다고 함. 나도 어느 정도 동의.

## 그래서 계획이 뭔가요?

- 만약 “IDI0에서 지정한 image data type이 아닌 모든 패킷의 정보가 두번째 버퍼로 빠진다”는 위 주장이 사실이라면, IDI0에 등록되지만 않았다면 `formats[]`에서 지원하는 image data type 중 어느 data type이든지 간에 우리의 metadata 전송용으로 활용할 수 있음.
    - 이렇게 하면 CSI-2 표준에서는 벗어나지만, Unicam 한정 메타데이터 전송이 가능할 것으로 보임.
    - 그런데 이걸 하기 전에 기본적으로 metadata 디코딩이 어떻게 일어나는 건지 궁금해서 더 찾아봤음.
- 원래대로라면 (근거 참고) Pixel Data 전후로 Frame start code가 short packet으로 들어온 이후, embedded data (ED) 임을 알리며 (Data Identifier 0x12) 패킷 한 줄이 들어옴.
    - 근거: CSI-2 Documentation: Sec 11.1.2 Embedded Information
    *Figure 57 - 프레임 구조 및 embedded data가 있어야 하는 위치를 정의하고 있음.
    본문 내용에 따르면 [ED]는 embedded data code, 즉 0x12를 data type으로 가지는 것으로 보임*
        
        ![IMG_2015.jpeg](%EC%9C%A0%EC%9D%BC%ED%95%9C%20%ED%95%B4%EA%B2%B0%EC%B1%85%20251af90d2677807684cfd8f2bd8c743a/IMG_2015.jpeg)
        
        > It is possible to embed extra lines containing addition information to the beginning and to the end of each picture frame as presented in the Figure 57. **If embedded information exists, then the lines containing the embedded data must use the embedded data code in the data identifier.**
        > 
- 각각의 Embedded data line에 어떤 내용이 기록되는지는 센서마다 다름. IMX219에서는 Data Format Code 0x0a 이후 tag, data, tag, data…가 반복되는 형태로 데이터를 기록하고 있음. 이 형식은 SMIA 표준의 Simplified 2-Byte Tagged Data Format에서 정의된 형식을 따르는 것으로 보임. raspiraw에서도 해당 방식으로 decoding하고 있음.
    - 근거 1: https://forums.raspberrypi.com/viewtopic.php?t=212558
        
        > Our current pitch calculation is
        > 
        > 
        > **Code: [Select all](https://forums.raspberrypi.com/viewtopic.php?t=212558#)**
        > 
        > ```c
        > pitch = VCOS_ALIGN_UP((port->format->es->video.crop.width  * bpp) >> 3, 16)
        > ```
        > 
        > but I can't guarantee that is going to apply to all sensors out there. **For embedded data, most start with 0x0A 0xAA as that comes from the SMIA embedded data format spec.** Searching for that is probably the best bet, but doesn't help for your optically black data.
        > 
    - 근거 2: https://github.com/raspberrypi/raspiraw/blob/master/raspiraw.c
        
        ```c
        void decodemetadataline(uint8_t *data, int bpp)
        {
        	int c = 1;
        	uint8_t tag, dta;
        	uint16_t reg = -1;
        
        	if (data[0] == 0x0a)  // Simplified 2-Byte Tagged Data Format을 의미
        	{
        
        		while (data[c] != 0x07)  // 0x07은 end-of-data tag임. paddding & invalid data에 대한 placeholder로 사용되기도 함
        		{
        			tag = data[c++];
        			if (bpp == 10 && (c % 5) == 4)
        				c++;
        			if (bpp == 12 && (c % 3) == 2)
        				c++;
        			dta = data[c++];
        
        			if (tag == 0xaa)  // 0xaa tag 뒤의 데이터가 CCI Register Index의 MSB를 담고 있음
        				reg = (reg & 0x00ff) | (dta << 8);
        			else if (tag == 0xa5)  // 0xa5 tag 뒤의 데이터가 CCI Register Index의 LSB를 담고 있음
        				reg = (reg & 0xff00) | dta;
        			else if (tag == 0x5a)  // 0x5a 뒤의 데이터는 valid data이니 파싱하고 다음 인덱스로 넘어가세요 (reg++)
        				vcos_log_error("Register 0x%04x = 0x%02x", reg++, dta);
        			else if (tag == 0x55)  // 0x55 뒤의 데이터는 invalid data (0x07)이니 파싱하지 말고 다음 인덱스로 넘어가세요 (reg++)
        				vcos_log_error("Skip     0x%04x", reg++);
        			else
        				vcos_log_error("Metadata decode failed %x %x %x", reg, tag, dta);
        		}
        	}
        	else
        		vcos_log_error("Doesn't looks like register set %x!=0x0a", data[0]);
        }
        ```
        
    - 근거 3: IMX219 Documentation: Sec. 4-1-6 CSI-2 Embedded Data Line
    *Fig. 24 - embedded data line은 프레임 앞쪽에 추가되고, 패킷 길이는 이미지 가로 길이와 같음
    Fig. 25, 26 - RAW10에서는 RAW8에서는 없는 Dummy Byte가 중간 중간에 생김
    Table 15 - 각각의 태그가 어떤 의미인지를 지정하고 있음*
        
        ![IMG_2016.jpeg](%EC%9C%A0%EC%9D%BC%ED%95%9C%20%ED%95%B4%EA%B2%B0%EC%B1%85%20251af90d2677807684cfd8f2bd8c743a/IMG_2016.jpeg)
        
        ![IMG_2017.jpeg](%EC%9C%A0%EC%9D%BC%ED%95%9C%20%ED%95%B4%EA%B2%B0%EC%B1%85%20251af90d2677807684cfd8f2bd8c743a/IMG_2017.jpeg)
        
    
    - 근거 4: SMIA Documentation 4.9.1: Simplified 2-Byte Tagged Data Format
    *IMX219 Documentation과 그림 및 표가 비슷해보이지만, 좀 더 자세하게 각각이 무슨 의미인지 알 수 있어서 참고하기 좋음
    Figure 31 - IMX219 Documentation Fig. 25, 26에 비해 조금 더 자세하게 설명하고 있음
    Table 35 (중요)
    - 우리가 이전에 0x0A 말고 본 적이 있는 Data Format Code는 0x00인데, 이건 Illegal Data를 의미함… 여태 우리는 뭘 받은거죠?
    - 0x0B부터 0xFE (Reserved for Future Use) 중에 하나를 선택해서 0x0A와 구분 가능하게 해야 함
    - 만약 second metadata buffer로 우리의 데이터가 들어왔을 때는 패킷 헤더가 날아간 상태로, payload밖에 없을 것임에 유의
    - 우리도 tag, data 반복 형식을 따를 것인지는 아직까지 미정이지만… 
       만약에 metadata가 valid한지 체크하는 로직이 있다면 이 형식을 따라야 하지 않을까 추측함
    - 또한 line width를 맞추기 위해 padding character 0x07을 사용하고 있는데, 우리는 어떻게 할 것인지도 정해야 함*
        
        ![IMG_2012.jpeg](%EC%9C%A0%EC%9D%BC%ED%95%9C%20%ED%95%B4%EA%B2%B0%EC%B1%85%20251af90d2677807684cfd8f2bd8c743a/IMG_2012.jpeg)
        
        ![IMG_2014.jpeg](%EC%9C%A0%EC%9D%BC%ED%95%9C%20%ED%95%B4%EA%B2%B0%EC%B1%85%20251af90d2677807684cfd8f2bd8c743a/IMG_2014.jpeg)
        
        ![IMG_2013.jpeg](%EC%9C%A0%EC%9D%BC%ED%95%9C%20%ED%95%B4%EA%B2%B0%EC%B1%85%20251af90d2677807684cfd8f2bd8c743a/IMG_2013.jpeg)
        
- IDI0 레지스터 값은 raspiraw의 imx219_modes.h 파일에서 RAW10에 해당하는 0x2B로 지정되어 있고, 바꾸고 싶다면 configuration file을 수정하거나, command line에서 레지스터 값을 직접 덮어쓸 수 있음.
    - 근거 1: https://github.com/raspberrypi/raspiraw/blob/master/imx219_modes.h
    *여기서 .image_id 값을 설정한다고 레지스터 값이 정말로 바뀌는지 궁금해할 수도 있는데
    - 여기서부터는 closed source라 알 수 없음.
    - `if (sensor_mode->image_id) rx_cfg.image_id = sensor_mode->image_id;`
    - 위처럼 `image_id`를 `MMAL_PARAMETER_CAMERA_RX_CONFIG_T` 자료형을 가진 변수 `rx_cfg`로 옮겨담는데
    - MMAL이랑 Unicam 둘 다 내부 코드는 proprietary라 못들여다 봄*
    
    ```c
    struct mode_def imx219_modes[] = {
    	{
    	    .regs = imx219_8MPix,
    	    .num_regs = NUM_ELEMENTS(imx219_8MPix),
    	    .width = 3280,
    	    .height = 2464,
    	    .encoding = 0,
    	    .order = BAYER_ORDER_BGGR,
    	    .native_bit_depth = 10,
    	    .image_id = 0x2B,  // 0x2B로 직접 지정
    	    .data_lanes = 2,
    	    .min_vts = 2504,
    	    .line_time_ns = 18904,
    	    .timing = { 0, 0, 0, 0, 0 },
    	    .term = { 0, 0 },
    	    .black_level = 66,
    	}, ...
    ```
    
    - 근거 2: https://github.com/raspberrypi/raspiraw/blob/master/raspiraw.c
        
        ```c
        	case CommandRegs: // register changes
        	{
        		len = strlen(argv[i + 1]);
        		cfg->regs = malloc(len + 1);
        		vcos_assert(cfg->regs);
        		strncpy(cfg->regs, argv[i + 1], len + 1);
        		i++;
        		break;
        	}
        
        ```
        
    - 근거 3: https://github.com/raspberrypi/raspiraw/tree/master
        
        **I2C register setting options**
        
        ```
        -r, --regs	: Change (current mode) regs
        ```
        
        Allows to change sensor regs in selected sensor mode. Format is a semicolon separated string of assignments. **An assignment consists of a 4 hex digits register address, followed by a comma and one or more 2 hex digit byte values. Restriction: Only registers present in selected sensor mode can be modified.** Example argument: "380A,003C;3802,78;3806,05FB". In case more than one byte appears after the comma, the byte values get written to next addresses.
        

## 요약하자면?

- **Virtual Channel은 (혹시 모르니) 0으로 설정하는 것이 좋을 듯.**
- **Data Type Value는 RGB888에 해당하는 0x24를 사용하는 것을 추천함.**
    - 0x12는 Trion T20에서 지원하지 않는다고 나와 있지만 속는 셈 치고 한번 해볼 수는 있음.
    - RGB888을 고른 이유는 일단 라즈베리파이에서 받을 수 있는 Data Type임이 확실해서임.
    - *추가적으로, YUV의 경우 legacy 버전이 존재하고 처리 로직이 나뉘어져 있는 것 같음. 불안해서 안 쓰는게 나을 것 같은데, 물론 문제없이 될 수도 있음!*
    - *그리고 잘 안된다면 0x20 - 0x23 중에서 번갈아가면서 시도해보는거 추천함. 코드 상으로는 IDI0에서 Image Data Type을 하나만 지정하는걸로 나와 있지만, 레지스터 크기가 32 bit라 최대 4개까지 저장할 수 있으므로 항상 4개씩 테스트해봐야 함. YUV Data Type도 혹시 모르니까 하나쯤은 테스트해볼 수 있음. 그닥 추천은 안함.*
- **Payload의 첫번째 byte는 0x0A와 구분 가능하게 해야 하고, 0x0B 추천함.**
    - 0x0B - 0xFE (Reserved for Future Use) 중에 하나를 선택하면 됨.
- **우리도 tag, data 반복 형식을 따를 것인지, 아니면 Word Count를 지정하고 사용할 것인지, 그리고 line width를 맞추기 위해 padding character를 사용할 것인지 정해야 함.**
    - *line width를 안 맞춰도 된다면 개꿀이고…*
    - 어차피 128 bit니까 고정된 Word Count만 파싱하고 나머지는 버리는 것도 방법이고
    - 헤더 형식을 우리 나름대로 지정해서, data[0] = 0x0B로 설정한 다음, 이후 data[1:2]에 Word Count를 지정할 수도 있음.
        - *fork한 raspiraw repo에서는 GPT-5 보고 이렇게 디코딩하라고 함.*
    - 참고로 Word Count를 지정하지 않을 거면 tag, data 반복하는 형식으로 해야 함
        - *padding placeholder (ex: 0x07)이 CMAC payload에서 나타나면 매우매우 곤란해지기 때문…*
- **IDI0 레지스터 값은 raspiraw에서 IMX219 기준 기본값이 0x2B로 지정되어 있으나, config 파일이나 command line interface로 자유롭게 변경할 수 있을 것으로 보임.**
    - *사실 이렇게까지 로우레벨로 동작하는 애인데 리눅스에서 제공하는 드라이버를 사용하기는 하는건지 의문이 들긴 함*
    - *애초에 Broadcom이 Unicam이랑 MMAL을 둘 다 만들었는데, 리눅스 드라이버를 굳이 쓰지 않아도 잘 돌아가게 되어 있지 않을까 싶음*
    - *하지만 Unicam에서 user-defined 8 bit (0x30-0x37)을 지원하지 않을 것임에는 동의함. 만약에 그게 지원이 된다면 드라이버에서도 해당 자료형을 받을 수 있게 설계했을 것으로 사료됨*

## Misc

- Timing issue 발생 가능성
    - 갑자기 걱정되는 부분이 우리가 스레드가 여러개인데, metadata가 두번째 버퍼로 아예 따로 들어오면 side channel 할 때랑 똑같이 타이밍 이슈가 생길 수도 있을 것 같음
    - 버퍼에 쓰기가 완료될 때마다 callback 함수가 호출되는데, 하나의 input feed 처리하기 위해서 스레드 여러 개가 달려든다면 상당히 곤란한 상황이 발생할 것으로 보임
    - 스레드 개수 한개로 제한하거나, 스레드가 여러개더라도 (이미 코드에 있는) frame count 변수에 더해 metadata count를 추가해서 핸들링해야 할 것으로 보임
    - 사실 별 이슈 아닐수도?
        - 근거 1: https://github.com/raspberrypi/raspiraw/issues/71
            
            vc.ril.rawcam will accept an buffers. It will write them with incoming data, and **return 2 buffers with the same timestamp** each time it receives a Frame End short packet on the CSI interface.
            
        - 근거 2: https://forums.raspberrypi.com/viewtopic.php?t=276047&utm_source=chatgpt.com
            
            **rawcam will only release the buffer when a new one has been returned to put the next buffer in.** It also fills pairs of buffers - one for the embedded metadata, and one for the image data.