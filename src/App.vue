<template>
<el-scrollbar id="app" style="height: 100%;">
  <div>
    <el-card header="通讯设置">
      <el-form>
        <el-form-item label="开关">
          <el-switch :disabled="!serial.name" v-model="serial.open" @change="open"></el-switch>
        </el-form-item>
        <el-form-item>
          <el-select :disabled="serial.port && serial.port.isOpen" v-model="serial.name" placeholder="端口号" @visible-change="visible => {if (visible) getPortData()}">
            <el-option v-for="item in serial.portData" :key="item.comName" :value="item.comName"></el-option>
          </el-select>
        </el-form-item>
        <!--
        <el-collapse>
          <el-collapse-item title="更多选项">
            <el-form-item>
              <el-select :disabled="serial.port && serial.port.isOpen" v-model="serial.baudRate" filterable allow-create placeholder="波特率">
                <el-option v-for="item in [110, 300, 1200, 2400, 4800, 9600, 14400, 19200, 38400, 57600, 115200]" :key="item" :value="item"></el-option>
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-select :disabled="serial.port && serial.port.isOpen" v-model="serial.dataBits" placeholder="数据位">
                <el-option v-for="item in [8, 7, 6, 5]" :key="item" :value="item"></el-option>
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-select :disabled="serial.port && serial.port.isOpen" v-model="serial.stopBits" placeholder="停止位">
                <el-option v-for="item in [1, 2]" :key="item" :value="item"></el-option>
              </el-select>
            </el-form-item>
            <el-form-item>
              <el-select :disabled="serial.port && serial.port.isOpen" v-model="serial.parity" filterable allow-create placeholder="校验位">
                <el-option v-for="item in [{ value: 'none', label: '无校验' }, { value: 'even', label: '偶校验' }, { value: 'odd', label: '奇校验' }]" :key="item.value" :label="item.label" :value="item.value"></el-option>
              </el-select>
            </el-form-item>
          </el-collapse-item>
        </el-collapse>
        -->
      </el-form>
    </el-card>
    <el-card header="设备列表" style="margin-top: 22px;">
      <div>
        <el-button type="primary" ref="refresh" icon="el-icon-refresh" @click="() => send('at+ln\r\n')"></el-button>
        <el-button type="primary" icon="el-icon-plus" @click="addDeviceDialog.open = true"></el-button>
        <el-button type="primary" icon="el-icon-setting" @click="editDialog.open = true"></el-button>
        <el-button type="danger" icon="el-icon-warning" @click="() => send('at+ureset\r\n')"></el-button>
      </div>
      <el-table :data="device.data" style="margin-top: 22px;">
        <el-table-column prop="address" label="地址"></el-table-column>
        <el-table-column prop="id" label="编号"></el-table-column>
        <el-table-column prop="name" label="名称"></el-table-column>
        <el-table-column label="状态"><template slot-scope="scope">{{scope.row.status == 1 ? '在线' : (scope.row.status == 2 ? '离线' : '')}}</template></el-table-column>
        <el-table-column width="200">
          <template slot-scope="scope">
            <el-button size="mini" round icon="el-icon-search" @click="() => {send('at+lv=' + scope.row.address + '\r\n')}"></el-button>
            <el-button size="mini" round type="primary" icon="el-icon-edit" @click="() => {device.p = scope.row, editAcDialog.open = true}"></el-button>
            <el-button size="mini" round type="danger" icon="el-icon-delete" @click="() => {send('at+dn=' + scope.row.address + '\r\n')}"></el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
    <el-card header="历史数据" style="margin-top: 22px;">
      <el-input type="textarea" v-model="log" :rows="4"></el-input>
    </el-card>
    <div>
      <el-dialog title="设置参数" :visible.sync="editDialog.open" width="80%" @closed="() => {$refs['set'].resetFields()}">
        <el-form ref="set" :model="editDialog.form">
          <el-form-item label="组网">
            <!--
            <el-switch v-model="editDialog.form.on" @change="on => {send('at+zjc=' + (on ? 'enable' : 'disable') + '\r\n')}"></el-switch>
            -->
            <el-button-group>
              <el-button @click="() => {send('at+zjc=enable\r\n')}">开</el-button>
              <el-button @click="() => {send('at+zjc=disable\r\n')}">关</el-button>
            </el-button-group>
          </el-form-item>
          <div style="border-bottom: 1px solid rgb(235, 238, 245); margin: -11px 0 11px 0;"></div>
          <!--
          <el-form-item>
            <el-select v-model="editDialog.form.role" filterable allow-create placeholder="角色">
              <el-option v-for="item in [{ value: 'c', label: '协调' }, { value: 'r', label: '路由' }, { value: 'e', label: '终端' }]" :key="item.value" :label="item.label" :value="item.value"></el-option>
            </el-select>
          </el-form-item>
          -->
          <el-form-item prop="panid" :rules="[{ required: true }, { type: 'number' }]">
            <el-input v-model.number="editDialog.form.panid" placeholder="编号"></el-input>
          </el-form-item>
          <el-form-item>
            <el-select v-model="editDialog.form.channel" filterable placeholder="信道">
              <el-option v-for="item in [11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26]" :key="item" :value="item"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item>
            <span>功率</span>
            <el-slider v-model="editDialog.form.power" :min="10" :max="20" :format-tooltip="v => v + 'dbm'" show-input></el-slider>
          </el-form-item>
          <el-form-item>
            <el-button @click="() => {send('at+zs?\r\n')}">读取</el-button>
            <el-button type="primary" @click="$refs['set'].validate(valid => {if (valid) send('at+zs=' + editDialog.form.role + ', ' + editDialog.form.panid + ', ' + editDialog.form.channel + ', ' + editDialog.form.power + '\r\n')})">设置</el-button>
          </el-form-item>
          <div style="border-bottom: 1px solid rgb(235, 238, 245); margin: -11px 0 11px 0;"></div>
          <el-form-item>
            <span>上报周期</span>
            <el-slider v-model="editDialog.reportInterval" :min="10" :max="86400" :format-tooltip="v => v + 's'" show-input @change="value => {send('at+ui=' + value + '\r\n')}"></el-slider>
          </el-form-item>
          <el-form-item>
            <el-button @click="() => {send('at+ui?\r\n')}">读取</el-button>
          </el-form-item>
          <el-form-item>
            <span>插座轮询</span>
            <el-slider v-model="editDialog.interval" :min="10" :max="3600" :format-tooltip="v => v + 's'" show-input @change="value => {send('at+pi=' + value + '\r\n')}"></el-slider>
          </el-form-item>
          <el-form-item>
            <el-button @click="() => {send('at+pi?\r\n')}">读取</el-button>
          </el-form-item>
        </el-form>
      </el-dialog>
      <el-dialog title="添加节点" :visible.sync="addDeviceDialog.open" width="80%" @closed="() => {$refs['add'].resetFields()}">
        <el-form ref="add" :model="addDeviceDialog.form">
          <el-form-item prop="address" required>
            <el-input v-model="addDeviceDialog.form.address" placeholder="地址"></el-input>
          </el-form-item>
          <el-form-item prop="id" required>
            <el-input v-model="addDeviceDialog.form.id" placeholder="编号"></el-input>
          </el-form-item>
          <el-form-item prop="name" required>
            <el-input v-model="addDeviceDialog.form.name" placeholder="名称"></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="$refs['add'].validate(valid => {if (valid) {send('at+an=' + addDeviceDialog.form.address + ', ' + addDeviceDialog.form.id + ', ' + addDeviceDialog.form.name + '\r\n'), addDeviceDialog.open = false}})">确定</el-button>
          </el-form-item>
        </el-form>
      </el-dialog>
      <el-dialog title="设置参数" :visible.sync="editAcDialog.open" width="80%">
        <el-form>
          <el-form-item>
            <!--
            <el-switch v-model="editAcDialog.on" @change="on => {send('at+acs=' + device.p.address + ', ' + (on ? 1 : 0) + '\r\n')}"></el-switch>
            -->
            <el-button-group>
              <el-button @click="() => {send('at+acs=' + device.p.address + ', 0\r\n')}">开</el-button>
              <el-button @click="() => {send('at+acs=' + device.p.address + ', 1\r\n')}">关</el-button>
            </el-button-group>
          </el-form-item>
          <el-form-item>
            <el-select v-model="editAcDialog.mode" clearable placeholder="模式" @change="value => {if (value !== '') send('at+acm=' + device.p.address + ', ' + value + '\r\n')}">
              <el-option v-for="item in [{ value: 0, label: '自动' }, { value: 1, label: '制冷' }, { value: 2, label: '制热' }, { value: 3, label: '抽风' }, { value: 4, label: '送风' }]" :key="item.value" :label="item.label" :value="item.value"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item>
            <span>温度</span>
            <el-slider v-model="editAcDialog.temperature" :min="16" :max="31" :format-tooltip="v => v + '°C'" @change="value => {send('at+act=' + device.p.address + ', ' + value + '\r\n')}"></el-slider>
          </el-form-item>
          <el-form-item>
            <el-select v-model="editAcDialog.speed" clearable placeholder="风速" @change="value => {if (value !== '') send('at+acwv=' + device.p.address + ', ' + value + '\r\n')}">
              <el-option v-for="item in [{ value: 0, label: '自动' }, { value: 1, label: '低风' }, { value: 2, label: '中风' }, { value: 3, label: '高风' }]" :key="item.value" :label="item.label" :value="item.value"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item>
            <el-select v-model="editAcDialog.direction" clearable placeholder="风向" @change="value => {if (value !== '') send('at+acwd=' + device.p.address + ', ' + value + '\r\n')}">
              <el-option v-for="item in [{ value: 0, label: '自动' }]" :key="item.value" :label="item.label" :value="item.value"></el-option>
            </el-select>
          </el-form-item>
        </el-form>
      </el-dialog>
    </div>
  </div>
</el-scrollbar>
</template>
<script>
  import { Message } from 'element-ui'
  export default {
    data() {
      return {
        serial: {
          open: false,
          port: null,
          name: null,
          baudRate: 9600,
          dataBits: 8,
          stopBits: 1,
          parity: 'none',
          portData: [],
          data: '',
          wait: false
        },
        device: {
          p: null,
          data: []
        },
        log: '',
        editDialog: {
          open: false,
          form: {
            role: 'c',
            panid: null,
            channel: 11,
            power: 0
          },
          reportInterval: 10,
          interval: 10
        },
        addDeviceDialog: {
          open: false,
          form: {
            address: null,
            id: null,
            name: null
          }
        },
        editAcDialog: {
          open: false,
          on: false,
          temperature: 16,
          speed: null,
          direction: null
        }
      }
    },
    methods: {
      getPortData() {
        this.SerialPort.list().then(portData => {this.serial.portData = portData})
      },
      open(on) {
        if (on) {
          this.serial.open = false
          if (this.serial.name) {
            this.serial.port = new this.SerialPort(this.serial.name, { autoOpen: false, baudRate: this.serial.baudRate, dataBits: this.serial.dataBits, stopBits: this.serial.stopBits, parity: this.serial.parity, highWaterMark: 1 })
            this.serial.port.on('data', this.onData)
            this.serial.port.on('open', () => {this.serial.open = true}), this.serial.port.on('close', () => {this.serial.port = null, this.serial.open = false})
            this.serial.port.on('error', error => {Message({ message: error.message, type: 'error' })})
            this.serial.port.open()
          }
        } else if (this.serial.port) {
          this.serial.port.close()
        }
      },
      send(data) {
        if (!this.serial.port) {
          Message({ message: '请先打开连接!', type: 'error' })
        } else if (this.serial.port.isOpen) {
          if (!this.serial.wait) {
            var i = 0
            var f = function () {
              this.serial.port.write(data[i], 'ascii')
              if (++i < data.length) {
                setTimeout(f, 100)
              } else {
                this.serial.wait = false
              }
            }.bind(this)
            this.serial.wait = 'aha'
            f()
            this.log += '[--->>>' + new Date().toJSON().replace('T', ' ').replace(/\..*$/, '') + ']' + data
          } else {
            Message({ message: '等一下!', type: 'warning' })
          }
        }
      },
      onData(data) {
        this.serial.data += data;
        var r;
        while (r = this.serial.data.match('(OK|Error)\r\n')) {
          data = this.serial.data.substring(0, r.index) + r[0];
          if (r[0].indexOf('OK') != -1) {
            if (data.indexOf('at+ln') != -1) {
              var d = data.split('\r\n')
              var deviceData = []
              for (var i = 0; i < d.length; i++) {
                var device
                if ((device = d[i].replace(/  /g, ' ').split(' ')).length == 4) {
                  deviceData.push({ name: device[0], address: device[1], id: device[2], status: device[3] })
                }
              }
              this.device.data = deviceData
            } else if (data.indexOf('at+dn') != -1 || data.indexOf('at+an') != -1) {
              Message({ message: '成功,请刷新设备列表!' })
            } else if (data.indexOf('at+zs?') != -1) {
              var d = data.split('\r\n')[1].match(/[0-9]+/g)
              {
                this.editDialog.form.panid = parseInt(d[0])
                this.editDialog.form.channel = parseInt(d[1])
                this.editDialog.form.power = parseInt(d[2])
              }
            } else if (data.indexOf('at+ui?') != -1) {
              this.editDialog.reportInterval = parseInt(data.split('\r\n')[1].replace('s', ''))
            } else if (data.indexOf('at+pi?') != -1) {
              this.editDialog.interval = parseInt(data.split('\r\n')[1].replace('s', ''))
            } else if (data.indexOf('AT+') == -1) {
              Message({ message: '成功!' })
            }
          } else {
            Message({ message: '失败!', type: 'error' })
          }
          this.log += '[<<<---' + new Date().toJSON().replace('T', ' ').replace(/\..*$/, '') + ']' + data
          this.serial.data = this.serial.data.substring(data.length)
        }
      }
    }
  }
</script>
<style>
  * {
    margin: 0;
    padding: 0;
  }
  html, body {
    height: 100%;
  }
  #app>.el-scrollbar__wrap {
    overflow-x: hidden;
  }
</style>