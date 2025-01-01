<template>
    <div class="wrapper">
		<div class="body">
			<div class="data-header"><span>环境数据</span></div>

			<div class="data-wrapper">
				<div class="data-temp">
					<img class="data-logo" src="/static/images/temp.png"/>
					<div class="data-text">
						<div class="data-title">温度</div>
						<div class="data-value">{{temp}}</div>
					</div>
				</div>
				<div class="data-humi">
					<img class="data-logo" src="/static/images/humi.png"/>
					<div class="data-text">
						<div class="data-title">湿度</div>
						<div class="data-value">{{humi}}</div>
					</div>
				</div>
			</div>


			<div class="data-header"><span>设备控制</span></div>

			<div class="data-wrapper">
				<div class="control-led">
					<img class="data-logo" src="/static/images/led.png"/>
					<div class="data-text">
						<div class="data-title">LED</div>
						<div class="data-value">
							<switch class="led-sw" @change="onLedChange" :checked="led" color="#4785f4"/>
						</div>
					</div>
				</div>
				<div class="control-alarm">
					<img class="data-logo" src="/static/images/alarm.png"/>
					<div class="data-text">
						<div class="data-title">报警器</div>
						<div class="data-value">
							<switch class="alarm-sw" @change="onAlarmChange" :checked="beep" color="#4785f4"/>
						</div>
					</div>
				</div>
			</div>

			<div class="data-header"><span>消息日志</span></div>

			<div class="data-wrapper">
				<div class="logsList">
					<ul class="container log-list">
					  <li v-for="(log, index) in logs.slice().reverse()" :key="index" class="log-item">
					    <card :text="log" :style="{ fontSize: '12px',  color: '#333' }"></card>
					  </li>
					</ul>
				</div>
			</div>
        </div>
    </div>
</template>

<script>
	import { connect } from '@/mqtt.js'
	import card from '@/card'
	const {
		createCommonToken
	} = require('@/key.js')
	export default {
		data() {
			return {
				client: {},
				temp: '1',
				humi: '1',
				led: true,
				beep: true,
				logs: []
			}
		},
		components: {
		    card
		},
		onLoad() {
			this.fetchDevData();
		},
		methods: {
			getFormattedDate() {
			  // 创建一个新的 Date 对象
			  const date = new Date();
			
			  // 获取年、月、日、小时和分钟
			  const year = date.getFullYear();
			  const month = (date.getMonth() + 1).toString().padStart(2, '0'); // 月份是从 0 开始的
			  const day = (date.getDate()-2).toString().padStart(2, '0');
			  const hours = (date.getHours()+10).toString().padStart(2, '0');
			  const minutes = date.getMinutes().toString().padStart(2, '0');
			  const seconds = date.getSeconds().toString().padStart(2, '0');
			  
			  // 组合成 yyyy-mm-dd hh:mm 格式
			  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}  `;
			},
			fetchDevData() {
				var _this = this
				_this.client = connect("ws://wf111287.ala.dedicated.aliyun.emqxcloud.cn:8083/mqtt", {
				            clientId: "wx",
				            username: "wx",
				            password: "123456"
				 });
				_this.logs.push(`${_this.getFormattedDate()} 成功连接设备`);
				_this.client.on("connect",function () {
				        console.log("成功连接 mqtt 服务器")
				        _this.client.subscribe("getData/3",function (error) {
				                if (!error) {
				                    console.log("成功订阅设备数据上行到小程序topic"),
									_this.logs.push(`${_this.getFormattedDate()} 成功订阅设备数据`);
				                }
				            }
				        )
				    }
				)
				
				_this.client.on(
				    "message",
				    function (topic, message) {
				        console.log('-----------------------------------------')
				        console.log('设备数据上行 topic:' + topic)
				        var sensorData = JSON.parse(message)
						
				        console.log(sensorData)
				        console.log(sensorData.Humi)
				        console.log(sensorData.Temp)
				        console.log(sensorData.Led)
				        console.log(sensorData.Beep)
				
				        //获取数据 绑定数据
				        _this.temp = sensorData.Temp + "℃"
				        _this.humi = sensorData.Humi + "%"
				        _this.led = sensorData.Led==1?true:false
						_this.beep = sensorData.Beep==1?true:false
						_this.logs.push(`${_this.getFormattedDate()}  设备数据: { Temp: ${_this.temp} , Humi: ${_this.humi} , Led: ${_this.led} , Beep: ${_this.beep} }`);
				        //_this.alarm = false
				    }
				)
			},
			//TODO 报警器开关执行方法
			onAlarmChange(event) {
			    var _this = this;
			    var switchState = event.mp.detail.value
			    console.log(switchState)
			    if (switchState) {
			        console.log("下发指令：开启报警")
			        _this.client.publish(
			            'Operate/3',
			            '{"Beep":1}',
			            function (error) {
			                if (!error) {
			                    console.log("成功下发指令到设备端：开启报警")
								_this.logs.push(`${_this.getFormattedDate()} 下发指令：开启报警`);
			                }
			            })
			    } else {
			        console.log("下发指令：关闭报警")
			        _this.client.publish(
			            'Operate/3',
			            '{"Beep":0}',
			            function (error) {
			                if (!error) {
			                    console.log("成功下发指令到设备端：关闭报警")
								_this.logs.push(`${_this.getFormattedDate()} 下发指令：关闭报警`);
			                }
			            })
			    }
			},
			// TODO LED开关执行方法
			onLedChange(event) {
			    var _this = this
			    var switchState = event.mp.detail.value
			    console.log(switchState)
			    if (switchState) {
			        console.log("下发指令：开灯")
			        _this.client.publish(
			            'Operate/3',
			            '{"LED":1}',
			            function (error) {
			                if (!error) {
			                    console.log("成功下发指令到设备端：开灯")
								_this.logs.push(`${_this.getFormattedDate()} 下发指令：开灯`);
			                }
			            })
			    } else {
			        console.log("下发指令：关灯")
			        _this.client.publish(
			            'Operate/3',
			            '{"LED":0}',
			            function (error) {
			                if (!error) {
			                    console.log("成功下发指令到设备端：关灯")
								_this.logs.push(`${_this.getFormattedDate()} 下发指令：关灯`);
			                }
			         })
			    }
			}
		}
	}
</script>


<style lang="scss" scoped>
.wrapper {
    padding: 12px;
    background-color: #f1f2f6;
    .btn {
        margin-top: 16px;
        border-radius: 10px;
        border: 2px solid #505050;
        color: #000000;
        background-color: #ffffff;
    }

    .header-wrapper {
        // background-color: #4285F4;
        background-color: #ffffff;
        border-radius: 20px;
        color: #000000;
        // box-shadow: #d6d6d6 0 0 5px;
        padding: 15px 15px;

        .header-title {
            display: flex;
            justify-content: space-between;
        }

        .header-text {
            font-size: 32px;
            font-weight: 400;
            display: flex;
            justify-content: space-between;
        }

        .weather-advice {
            margin-top: 20px;
            font-size: 12px;
        }
    }

    .data-wrapper {
        margin-top: 8px;
        display: flex;
        justify-content: space-between;


        .data-temp {
            background-color: #fdefc2;
            width: 49%;
			height: 120px;
			border-radius: 20px;
			display: flex;
			justify-content: space-around;
			// padding: 0 10px;

			.led-sw{
				margin-top: 7px;
				// margin-right: -10px;
			}

			.data-logo {
				height: 50px;
				width: 50px;
				margin-top: 35px;
			}

			.data-text {
				margin-top: 29px;
				color: #202020;

				.data-title {
					text-align: center;
					font-size: 17px;
				}

				.data-value {
					font-size: 26px;
				}
			}
        }

        .data-humi {
            background-color: #d2e4fc;
            width: 49%;
			height: 120px;
			border-radius: 20px;
			display: flex;
			justify-content: space-around;
			// padding: 0 10px;

			.led-sw{
				margin-top: 7px;
				// margin-right: -10px;
			}

			.data-logo {
				height: 50px;
				width: 50px;
				margin-top: 35px;
			}

			.data-text {
				margin-top: 29px;
				color: #202020;

				.data-title {
					text-align: center;
					font-size: 17px;
				}

				.data-value {
					font-size: 26px;
				}
			}
        }


        .control-led {
            background-color: #d2e4fc;
            width: 49%;
            height: 120px;
            border-radius: 20px;
            display: flex;
            justify-content: space-around;
            // padding: 0 10px;

            .led-sw{
                margin-top: 6px;
                // margin-right: -10px;
            }

            .data-logo {
                height: 50px;
                width: 50px;
                margin-top: 35px;
            }

            .data-text {
                margin-top: 29px;
                color: #202020;

                .data-title {
                    text-align: center;
					font-size: 17px;
                }

                .data-value {
                    font-size: 26px;
                }
            }
        }

        .control-alarm {
            background-color: #fdd1cc;
            width: 49%;
			height: 120px;
			border-radius: 20px;
			display: flex;
			justify-content: space-around;
			// padding: 0 10px;

			.alarm-sw{
				margin-top: 6px;
				// margin-right: -10px;
			}

			.data-logo {
				height: 50px;
				width: 50px;
				margin-top: 35px;
			}

			.data-text {
				margin-top: 29px;
				color: #202020;

				.data-title {
					text-align: center;
					font-size: 17px;
				}

				.data-value {
					font-size: 26px;
				}
			}
        }


    }

    .aqi-span{
        font-size: 20px;
    }

    .weather-header {
        margin-left: 5px;
        // margin-top: 5px;
        margin-right: 10px;
        margin-bottom: 10px;
        font-size: 20px;
    }

    .data-header {
        margin-left: 5px;
        margin-top: 15px;
		margin-down: 30px;
        margin-right: 10px;
        // margin-bottom: 5px;
        font-size: 20px;
    }

    //.divide-line {
    //    margin-top: 20px;
    //
    //    background: #e0e3da;
    //    width: 50%;
    //    height: 2px;
    //    display: flex;
    //    flex-direction: row;
    //
    //    justify-content: center;
    //}

    .divide-line {
        margin-top: 15px;
        background: #000000;
        width: 100%;
        height: 1px;
    }


}

.logsList {
	overflow: auto;
    background-color: #d2e4fc;
    width: 100%;
    height: 300px;
    border-radius: 20px;
    display: flex;
    padding: 10px 20px 20px 20px;
    .data-text {
        margin: 15px;
        color: #202020;

        .data-value {
            font-size: 12px;
            // text-align: left;
        }
    }
}

.log-list {
    display: flex;
    flex-direction: column;
    padding: 2px;
}

.log-item {
    // margin: 10px;
    color: #202020;
    text-align: left;
    margin: 2px;
    font-size: 12px;
}

</style>
