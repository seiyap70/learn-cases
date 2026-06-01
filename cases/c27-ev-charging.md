# C27: 新能源车充电桩的智能调度

## 业务场景

某新能源车充电运营平台，管理 1 万个充电桩，分布在 500 个充电站。充电桩是重资产——利用率直接决定盈利能力。但充电桩不是"有空位就能用"——它受电网功率约束。

**已知数据：**
- 充电桩数：1 万个（快充 60kW 30%，慢充 7kW 70%）
- 日均充电次数：5 万次
- 峰值同时充电：8000 桩
- 充电时长：快充 30-60 分钟，慢充 4-8 小时
- 单次充电费用：30-80 元
- 电网约束：单个充电站最大功率 2MW

**为什么是难题？**

一个站有 20 个 60kW 快充桩，总功率需求 1.2MW，但站点最大功率只有 2MW。最多同时 30 个桩快充（1.8MW），超过则跳闸。如果 15 个桩已经在快充，第 16 个用户来了怎么办？——降功率充电（60kW → 30kW），还是排队等候？

## 核心挑战

### 挑战 1：功率约束下的充电调度

站内总功率有上限，不能所有桩同时满功率充电。需要动态分配功率。核心矛盾：用户期望满功率快充 vs 电网功率有限。调度策略不仅影响单个用户体验，还影响全站吞吐——如果让低电量车快充、高电量车降功率，整体充电效率更高。

### 挑战 2：预约与实际使用的偏差

用户预约了 10:00-11:00 的快充桩，但 10:30 才到 → 桩空闲 30 分钟（利用率损失）。充完不挪车 → 后面预约的用户无法使用。更棘手的是：多个用户预约同一时段同一桩，需要完整的调度和冲突处理。

### 挑战 3：电价波动下的充电优化

分时电价：谷时 ¥0.3/度，峰时 ¥1.2/度。4 倍价差意味着同样充 50 度电，谷时 ¥15 vs 峰时 ¥60。如何引导用户谷时充电，提升谷时利用率？这不只是展示价格——需要设计完整的激励机制和调度策略。

### 挑战 4：设备状态实时监控

充电桩是 IoT 设备，通过 MQTT 上报状态。1 万个桩每 10 秒上报一次 = 1000 次/秒。需要实时监控各桩状态（空闲/充电中/故障/离线），并据此调度。桩故障时需要自动切换到备用桩并通知用户。

## 设计约束

- 不超过站点功率上限（跳闸 = 全站断电 = 严重事故）
- 预约宽限期 15 分钟
- 用户等待排队时间 < 30 分钟

## 请先独立思考（限时 30 分钟）

1. 功率调度算法：如何在功率预算内分配各桩的充电功率？降功率 vs 排队？
2. 预约系统：如何处理迟到和不挪车？宽限期如何设计？
3. 电价引导：如何设计激励机制让用户选择谷时充电？

---

## 设计解析

### 数据库设计：充电桩与站点

```sql
-- 充电站
CREATE TABLE charging_stations (
    station_id VARCHAR(32) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    address VARCHAR(200),
    lat DECIMAL(10,7) NOT NULL,
    lng DECIMAL(10,7) NOT NULL,
    max_power_kw INT NOT NULL,          -- 站点最大功率（kW）
    total_piles INT NOT NULL,            -- 总桩数
    status VARCHAR(20) DEFAULT 'active', -- active / maintenance / offline
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_location (lat, lng)
);

-- 充电桩
CREATE TABLE charging_piles (
    pile_id VARCHAR(32) PRIMARY KEY,
    station_id VARCHAR(32) NOT NULL,
    pile_type VARCHAR(10) NOT NULL,      -- fast(60kW) / slow(7kW)
    max_power_kw INT NOT NULL,           -- 桩最大功率
    current_power_kw INT DEFAULT 0,      -- 当前充电功率（动态更新）
    status VARCHAR(20) DEFAULT 'idle',   -- idle / charging / derated / fault / offline
    mqtt_topic VARCHAR(100),             -- 设备MQTT主题
    last_heartbeat TIMESTAMP,
    
    INDEX idx_station (station_id),
    INDEX idx_status (status),
    FOREIGN KEY (station_id) REFERENCES charging_stations(station_id)
);

-- 预约记录
CREATE TABLE reservations (
    reservation_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(32) NOT NULL,
    pile_id VARCHAR(32) NOT NULL,
    station_id VARCHAR(32) NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    grace_end TIMESTAMP NOT NULL,         -- 宽限期截止 = start_time + 15min
    duration_min INT NOT NULL,
    status VARCHAR(20) DEFAULT 'reserved', -- reserved / charging / completed / expired / cancelled
    actual_start_time TIMESTAMP,           -- 用户实际到达时间
    actual_end_time TIMESTAMP,             -- 充电实际结束时间
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_user (user_id),
    INDEX idx_pile_time (pile_id, start_time, end_time),
    INDEX idx_status_time (status, start_time)
);

-- 充电订单
CREATE TABLE charge_orders (
    order_id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(32) NOT NULL,
    pile_id VARCHAR(32) NOT NULL,
    reservation_id VARCHAR(64),
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    power_kw INT NOT NULL,                -- 实际充电功率
    energy_kwh DECIMAL(8,2) DEFAULT 0,    -- 充电量
    cost DECIMAL(10,2) DEFAULT 0,          -- 总费用
    electricity_cost DECIMAL(10,2) DEFAULT 0, -- 电费
    service_cost DECIMAL(10,2) DEFAULT 0,    -- 服务费
    occupancy_fee DECIMAL(10,2) DEFAULT 0,   -- 占位费
    charge_status VARCHAR(20) DEFAULT 'charging', -- charging / complete / occupancy / closed
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_pile (pile_id),
    INDEX idx_user_time (user_id, start_time)
);

-- 功率调度日志（审计）
CREATE TABLE power_schedule_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    station_id VARCHAR(32) NOT NULL,
    pile_id VARCHAR(32) NOT NULL,
    action VARCHAR(20) NOT NULL,          -- approve / derate / queue / upgrade / release
    requested_power_kw INT,
    actual_power_kw INT,
    total_power_before INT,               -- 调度前站点总功率
    total_power_after INT,                -- 调度后站点总功率
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_station_time (station_id, created_at)
);
```

### 充电桩状态机

```
桩状态流转：
  idle ───→ charging ───→ complete ───→ idle（用户挪车）
    │           │              │
    │           ↓              ↓
    │       derated        occupancy（充完不挪车）
    │           │              │
    │           ↓              ↓
    │       charging       occupancy → closed（收取占位费后强制释放）
    │           │
    ↓           ↓
  fault ───→ offline ───→ idle（恢复后）
```

状态变更触发条件：

| 当前状态 | 事件 | 目标状态 | 夦理 |
|---------|------|---------|------|
| idle | 用户扫码/预约到达 | charging | 分配功率，开始计费 |
| idle | 功率不足 | derated | 降功率充电，通知用户 |
| charging | 充满/用户停止 | complete | 停止计费，提醒挪车 |
| charging | 功率升级 | charging | 功率调高，通知用户 |
| complete | 10分钟未挪车 | occupancy | 开始收占位费 |
| charging/derated | 设备故障 | fault | 停止充电，通知用户换桩 |
| 任意 | 网络断开 | offline | 标记离线，暂停调度 |
| fault | 维修完成 | idle | 恢复可用 |
| offline | 网络恢复 | idle | 恢复可用 |

### MQTT 通信协议

```python
class MQTTProtocol:
    """充电桩与云端的MQTT通信协议"""

    # 消息主题定义
    TOPICS = {
        # 上行：桩 → 云端
        "state_report":   "pile/{pile_id}/state",        # 状态上报（每10秒）
        "heartbeat":      "pile/{pile_id}/heartbeat",    # 心跳（每30秒）
        "charge_progress": "pile/{pile_id}/progress",    # 充电进度（每30秒）
        "charge_complete": "pile/{pile_id}/complete",    # 充电完成
        "fault_report":   "pile/{pile_id}/fault",        # 故障上报
        "cmd_ack":        "pile/{pile_id}/ack",          # 指令确认

        # 下行：云端 → 桩
        "start_charge":   "cmd/{pile_id}/start",         # 开始充电
        "stop_charge":    "cmd/{pile_id}/stop",          # 停止充电
        "set_power":      "cmd/{pile_id}/power",         # 设置充电功率
        "reset":          "cmd/{pile_id}/reset",         # 重启桩
    }

    def on_state_report(self, pile_id, message):
        """处理桩状态上报"""
        state = json.loads(message)
        # state: {"status":"charging","power_kw":60,"energy_kwh":15.3,
        #          "voltage":380,"current":158,"temperature":42}

        # 更新桩状态到缓存和数据库
        self.state_cache.update(pile_id, state)
        self.db.update("charging_piles", {
            "status": state["status"],
            "current_power_kw": state["power_kw"],
            "last_heartbeat": now()
        }, {"pile_id": pile_id})

        # 如果功率变化 → 检查是否需要调整其他桩的功率
        if state["status"] == "charging":
            self.power_scheduler.rebalance(self.get_station_id(pile_id))

    def on_charge_progress(self, pile_id, message):
        """处理充电进度上报"""
        progress = json.loads(message)
        # progress: {"energy_kwh":15.3,"soc":45,"estimated_end":"2024-05-07T15:30:00"}

        # 更新充电订单
        self.db.update("charge_orders", {
            "energy_kwh": progress["energy_kwh"]
        }, {"pile_id": pile_id, "charge_status": "charging"})

        # 推送给用户App
        self.push_to_user(pile_id, progress)

    def on_heartbeat(self, pile_id, message):
        """心跳处理"""
        hb = json.loads(message)
        self.state_cache.update_heartbeat(pile_id)

        # 检测离线：心跳超过60秒未上报 → 标记offline
        # 由独立的健康检查线程处理

    def send_start_charge(self, pile_id, power_kw):
        """下发充电指令"""
        cmd = {
            "command": "start",
            "power_kw": power_kw,
            "request_id": uuid4(),
            "timestamp": now_ms()
        }
        self.mqtt.publish(f"cmd/{pile_id}/start", json.dumps(cmd))

    def send_set_power(self, pile_id, power_kw):
        """下发功率调整指令"""
        cmd = {
            "command": "set_power",
            "power_kw": power_kw,
            "request_id": uuid4(),
            "timestamp": now_ms()
        }
        self.mqtt.publish(f"cmd/{pile_id}/power", json.dumps(cmd))
```

**MQTT QoS 选择：**

| 消息类型 | QoS | 原因 |
|---------|-----|------|
| 心跳 | 0 | 丢了无所谓，下次就来了 |
| 状态上报 | 1 | 至少到达一次 |
| 充电指令 | 2 | 必须到达且只到达一次（安全相关） |
| 充电完成 | 2 | 必须到达（计费相关） |
| 功率调整 | 1 | 至少到达一次 |

### OCPP 协议处理器

充电桩与云端的通信不仅依赖 MQTT，生产环境中通常还需要实现 OCPP（Open Charge Point Protocol）协议，这是充电桩行业的国际标准。OCPP 2.0.1 定义了充电桩与中央系统之间的完整通信规范。

```python
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Dict, List
from datetime import datetime
import json
import uuid
import logging

logger = logging.getLogger(__name__)


class OCPPMessageAction(Enum):
    """OCPP 2.0.1 标准动作定义"""
    # CSMS → Charging Station（下行）
    REQUEST_START_TRANSACTION = "RequestStartTransaction"
    REQUEST_STOP_TRANSACTION = "RequestStopTransaction"
    SET_VARIABLE = "SetVariable"
    GET_VARIABLE = "GetVariable"
    REMOTE_START = "RemoteStartTransaction"
    REMOTE_STOP = "RemoteStopTransaction"
    RESET = "Reset"
    UNLOCK_CONNECTOR = "UnlockConnector"
    TRIGGER_MESSAGE = "TriggerMessage"
    UPDATE_FIRMWARE = "UpdateFirmware"

    # Charging Station → CSMS（上行）
    BOOT_NOTIFICATION = "BootNotification"
    HEARTBEAT = "Heartbeat"
    AUTHORIZE = "Authorize"
    START_TRANSACTION = "StartTransaction"
    STOP_TRANSACTION = "StopTransaction"
    METER_VALUES = "MeterValues"
    STATUS_NOTIFICATION = "StatusNotification"
    ALARM_SIGNAL = "AlarmSignal"
    FIRMWARE_STATUS = "FirmwareStatusNotification"
    LOG_STATUS = "LogStatusNotification"


class ChargePointStatus(Enum):
    """充电桩连接器状态"""
    AVAILABLE = "Available"
    OCCUPIED = "Occupied"
    RESERVED = "Reserved"
    UNAVAILABLE = "Unavailable"
    FAULTED = "Faulted"


class ChargingProfileKind(Enum):
    """充电策略类型"""
    ABSOLUTE = "Absolute"        # 绝对时间
    RECURRING = "Recurring"      # 周期性
    RELATIVE = "Relative"        # 相对时间


@dataclass
class ChargingSchedulePeriod:
    """充电计划时段"""
    start_period: int            # 从计划开始的秒数
    limit: float                 # 功率限制（安培或千瓦）
    number_phases: int = 3       # 相数


@dataclass
class ChargingSchedule:
    """充电计划"""
    duration: int                              # 计划总时长（秒）
    charging_rate_unit: str = "W"              # 充电速率单位：W / A
    periods: List[ChargingSchedulePeriod] = field(default_factory=list)
    min_charging_rate: float = 0.0             # 最低充电速率
    start_schedule: Optional[datetime] = None  # 计划开始时间


@dataclass
class ChargingProfile:
    """充电策略配置"""
    profile_id: int
    stack_level: int                           # 优先级（越高越优先）
    charging_profile_purpose: str              # ChargingStationMaxProfile / TxProfile
    charging_profile_kind: ChargingProfileKind
    schedule: ChargingSchedule
    valid_from: Optional[datetime] = None
    valid_to: Optional[datetime] = None
    transaction_id: Optional[str] = None


class OCPPProtocolHandler:
    """OCPP 2.0.1 协议处理器——管理充电桩与云端的完整通信"""

    def __init__(self, station_repo, order_repo, power_scheduler, event_bus):
        self.station_repo = station_repo
        self.order_repo = order_repo
        self.power_scheduler = power_scheduler
        self.event_bus = event_bus
        self.pending_requests: Dict[str, dict] = {}   # request_id → callback
        self.charge_points: Dict[str, dict] = {}       # pile_id → 连接信息
        self._boot_cache: Dict[str, dict] = {}         # pile_id → BootNotification缓存

    # ========== 上行消息处理：充电桩 → 云端 ==========

    async def handle_boot_notification(self, pile_id: str, payload: dict) -> dict:
        """
        处理充电桩启动通知
        桩上电后第一条消息，包含设备型号、固件版本等
        """
        logger.info(f"BootNotification from {pile_id}: {payload}")

        boot_info = {
            "pile_id": pile_id,
            "charge_point_model": payload.get("chargePointModel"),
            "charge_point_vendor": payload.get("chargePointVendor"),
            "charge_point_serial": payload.get("chargePointSerialNumber"),
            "firmware_version": payload.get("firmwareVersion"),
            "meter_type": payload.get("meterType"),
            "meter_serial": payload.get("meterSerialNumber"),
            "booted_at": datetime.utcnow(),
        }
        self._boot_cache[pile_id] = boot_info

        # 更新数据库中桩的固件版本信息
        self.station_repo.update_pile_info(pile_id, {
            "firmware_version": boot_info["firmware_version"],
            "serial_number": boot_info["charge_point_serial"],
            "status": "idle",
            "last_heartbeat": datetime.utcnow(),
        })

        # 检查是否有待执行的固件更新
        pending_fw = self.station_repo.get_pending_firmware(pile_id)

        # 返回配置：心跳间隔、时间同步等
        response = {
            "currentTime": datetime.utcnow().isoformat(),
            "interval": 30,                     # 心跳间隔30秒
            "status": "Accepted",
        }

        if pending_fw:
            # 启动后延迟5分钟推送固件更新，避免影响初始充电
            self.event_bus.schedule_delayed(
                delay_seconds=300,
                action="push_firmware",
                data={"pile_id": pile_id, "firmware": pending_fw}
            )

        self.event_bus.emit("pile_booted", pile_id=pile_id, boot_info=boot_info)
        return response

    async def handle_heartbeat(self, pile_id: str, payload: dict) -> dict:
        """
        处理心跳
        桩定期上报，用于检测在线状态
        """
        # 更新心跳时间戳（写入Redis缓存，高速写入）
        self.station_repo.update_heartbeat(pile_id)

        # 返回当前时间（用于桩端时间同步校准）
        return {
            "currentTime": datetime.utcnow().isoformat()
        }

    async def handle_authorize(self, pile_id: str, payload: dict) -> dict:
        """
        处理充电授权请求
        用户扫码/刷卡后，桩请求云端验证用户身份和支付能力
        """
        id_token = payload.get("idToken", {})
        token_type = id_token.get("type")    # RFID / Central / eMAID / ISO14443
        token_value = id_token.get("idToken")

        # 验证用户身份
        user = self.order_repo.find_user_by_token(token_type, token_value)

        if not user:
            return {"idTokenInfo": {"status": "Invalid"}}

        # 检查用户是否有未支付的订单（信用控制）
        unpaid_orders = self.order_repo.get_unpaid_orders(user["user_id"])
        if unpaid_orders and len(unpaid_orders) >= 3:
            return {"idTokenInfo": {"status": "ConcurrentTx"}}

        # 检查用户是否有进行中的充电会话（防止一卡多充）
        active_sessions = self.order_repo.get_active_sessions(user["user_id"])
        if active_sessions:
            return {"idTokenInfo": {"status": "ConcurrentTx"}}

        # 检查用户支付方式（预授权/后付费）
        payment_method = self.order_repo.get_payment_method(user["user_id"])

        # 如果是预付费模式，检查余额是否充足（预估充电费用）
        if payment_method["type"] == "prepaid":
            estimated_cost = 80.0  # 预估最高费用
            if payment_method["balance"] < estimated_cost * 0.3:
                return {"idTokenInfo": {"status": "NotAllowed"}}

        # 授权通过
        return {
            "idTokenInfo": {
                "status": "Accepted",
                "cacheExpiryDateTime": (datetime.utcnow().isoformat()),  # 授权缓存有效期
                "groupIdToken": {"idToken": user["user_id"]},
            }
        }

    async def handle_start_transaction(self, pile_id: str, payload: dict) -> dict:
        """
        处理充电会话开始
        桩确认开始充电后上报，包含用户ID、连接器编号等
        """
        connector_id = payload.get("connectorId", 1)
        id_token = payload.get("idToken", {})
        meter_start = payload.get("meterStart", 0)    # 起始电表读数（Wh）
        timestamp = payload.get("timestamp")

        user_id = id_token.get("idToken")

        # 向功率调度器请求充电功率
        pile_info = self.station_repo.get_pile(pile_id)
        schedule_result = self.power_scheduler.request_charge(
            station_id=pile_info["station_id"],
            pile_id=pile_id,
            requested_power_kw=pile_info["max_power_kw"],
            priority="normal"
        )

        if schedule_result["status"] == "queued":
            # 无可用功率，拒绝本次充电
            return {
                "transactionId": "",
                "idTokenInfo": {"status": "NotAllowed"},
                "chargingSchedule": None,
            }

        # 创建充电订单
        transaction_id = str(uuid.uuid4())
        order = {
            "order_id": transaction_id,
            "user_id": user_id,
            "pile_id": pile_id,
            "connector_id": connector_id,
            "meter_start_wh": meter_start,
            "start_time": datetime.utcnow(),
            "power_kw": schedule_result.get("power_kw", pile_info["max_power_kw"]),
            "charge_status": "charging",
        }
        self.order_repo.create_order(order)

        # 构建充电策略（约束功率）
        charging_profile = None
        if schedule_result["status"] == "derated":
            actual_power = schedule_result["power_kw"]
            charging_profile = self._build_power_limit_profile(
                transaction_id=transaction_id,
                max_power_watts=int(actual_power * 1000),
            )

        # 更新桩状态
        self.station_repo.update_pile(pile_id, {
            "status": "charging" if schedule_result["status"] == "approved" else "derated",
            "current_power_kw": schedule_result.get("power_kw", 0),
        })

        # 发布事件
        self.event_bus.emit("transaction_started", transaction_id=transaction_id,
                           pile_id=pile_id, user_id=user_id,
                           power_kw=schedule_result.get("power_kw"))

        response = {
            "transactionId": transaction_id,
            "idTokenInfo": {"status": "Accepted"},
        }
        if charging_profile:
            response["chargingSchedule"] = charging_profile

        return response

    async def handle_stop_transaction(self, pile_id: str, payload: dict) -> dict:
        """
        处理充电会话结束
        桩上报充电结束，包含最终电表读数、停止原因等
        """
        transaction_id = payload.get("transactionId")
        meter_stop = payload.get("meterStop", 0)     # 结束电表读数（Wh）
        timestamp = payload.get("timestamp")
        reason = payload.get("reason", "EVDisconnected")
        # reason: EVDisconnected / EnergyLimitReached / SOCLimitReached / StoppedByEV /
        #         LocalStop / RemoteStop / PowerLoss / Reboot / DeAuthorized / Other

        # 获取订单信息
        order = self.order_repo.get_order(transaction_id)
        if not order:
            logger.error(f"StopTransaction: order {transaction_id} not found")
            return {"idTokenInfo": {"status": "Invalid"}}

        # 计算充电量
        energy_wh = meter_stop - order.get("meter_start_wh", 0)
        energy_kwh = max(0, energy_wh / 1000.0)

        # 计算充电费用（调用计费模块）
        billing_result = self.billing_service.calculate_charge_cost(
            order_id=transaction_id,
            energy_kwh=energy_kwh,
            start_time=order["start_time"],
            end_time=datetime.utcnow(),
            power_kw=order.get("power_kw", 0),
        )

        # 更新订单
        self.order_repo.update_order(transaction_id, {
            "end_time": datetime.utcnow(),
            "energy_kwh": energy_kwh,
            "cost": billing_result["total_cost"],
            "electricity_cost": billing_result["electricity_cost"],
            "service_cost": billing_result["service_cost"],
            "charge_status": "complete",
            "stop_reason": reason,
        })

        # 释放功率预算
        self.power_scheduler.on_charge_complete(pile_id)

        # 更新桩状态
        self.station_repo.update_pile(pile_id, {
            "status": "idle",
            "current_power_kw": 0,
        })

        # 启动占位超时检测（10分钟后开始收占位费）
        self.event_bus.schedule_delayed(
            delay_seconds=600,
            action="check_occupancy",
            data={"pile_id": pile_id, "user_id": order["user_id"],
                  "transaction_id": transaction_id}
        )

        # 推送充电完成通知
        self.event_bus.emit("transaction_stopped",
                           transaction_id=transaction_id,
                           pile_id=pile_id,
                           user_id=order["user_id"],
                           energy_kwh=energy_kwh,
                           cost=billing_result["total_cost"])

        return {"idTokenInfo": {"status": "Accepted"}}

    async def handle_meter_values(self, pile_id: str, payload: dict) -> dict:
        """
        处理电表数值上报
        桩定期上报电压、电流、功率、SOC等
        """
        transaction_id = payload.get("transactionId")
        meter_values = payload.get("meterValue", [])

        for mv in meter_values:
            timestamp = mv.get("timestamp")
            sampled_values = mv.get("sampledValue", [])

            for sv in sampled_values:
                measurand = sv.get("measurand", "Energy.Active.Import.Register")
                value = sv.get("value")
                unit = sv.get("unit", "Wh")
                context = sv.get("context", "Sample.Periodic")

                # 根据测量类型处理
                if measurand == "Energy.Active.Import.Register":
                    # 电量累计值 → 更新订单充电量
                    energy_kwh = float(value) / 1000.0
                    self.order_repo.update_order_progress(
                        transaction_id, energy_kwh)

                elif measurand == "Power.Active.Import":
                    # 实时功率 → 用于功率调度判断
                    power_kw = float(value) / 1000.0
                    self.station_repo.update_pile_power(pile_id, power_kw)

                elif measurand == "SoC":
                    # 电池SOC → 推送给用户 + 用于调度决策
                    soc = float(value)
                    self.event_bus.emit("soc_update",
                                       pile_id=pile_id,
                                       transaction_id=transaction_id,
                                       soc=soc)
                    # SOC > 80% 时可考虑降功率（锂电特性：80%以上充电效率急剧下降）
                    if soc > 80:
                        self._handle_high_soc(pile_id, transaction_id, soc)

                elif measurand == "Temperature":
                    # 温度监控 → 过温保护
                    temp = float(value)
                    if temp > 85:  # 充电桩内部温度超过85°C
                        self._handle_over_temperature(pile_id, temp)

        return {}

    async def handle_status_notification(self, pile_id: str, payload: dict) -> dict:
        """
        处理桩状态变更通知
        """
        connector_id = payload.get("connectorId", 1)
        status = payload.get("status")           # Available / Occupied / Reserved / Unavailable / Faulted
        error_code = payload.get("errorCode", "NoError")
        info = payload.get("info", "")
        timestamp = payload.get("timestamp")

        status_mapping = {
            "Available": "idle",
            "Occupied": "charging",
            "Reserved": "reserved",
            "Unavailable": "offline",
            "Faulted": "fault",
        }
        mapped_status = status_mapping.get(status, "offline")

        # 更新桩状态
        self.station_repo.update_pile(pile_id, {
            "status": mapped_status,
        })

        # 如果是故障状态，触发故障处理
        if mapped_status == "fault":
            self.event_bus.emit("pile_fault",
                               pile_id=pile_id,
                               error_code=error_code,
                               info=info)
        elif mapped_status == "idle":
            # 桩变为空闲 → 检查排队队列
            pile_info = self.station_repo.get_pile(pile_id)
            self.power_scheduler.process_wait_queue(pile_info["station_id"])

        return {}

    async def handle_alarm_signal(self, pile_id: str, payload: dict) -> dict:
        """
        处理报警信号
        OCPP 2.0.1 新增：桩主动上报紧急报警
        """
        alarm_type = payload.get("alarmType")
        # alarm_type: OverTemperature / OverCurrent / OverVoltage /
        #             UnderVoltage / GroundFault / CPSignalFailure

        logger.critical(f"ALARM from {pile_id}: {alarm_type}")

        # 立即降功率或停止充电
        if alarm_type in ("OverTemperature", "OverCurrent", "GroundFault"):
            # 安全类报警：立即停止充电
            await self._emergency_stop(pile_id, reason=f"safety_alarm:{alarm_type}")
        elif alarm_type in ("OverVoltage", "UnderVoltage"):
            # 电压异常：降低功率
            self.power_scheduler.derate_pile(pile_id, factor=0.5)

        self.event_bus.emit("pile_alarm", pile_id=pile_id, alarm_type=alarm_type)
        return {}

    async def handle_firmware_status(self, pile_id: str, payload: dict) -> dict:
        """
        处理固件更新状态上报
        """
        status = payload.get("status")
        # status: Downloaded / DownloadFailed / Downloading / Idle / Installing /
        #         InstallFailed / InstallScheduled / InstallRebooting

        logger.info(f"Firmware status for {pile_id}: {status}")

        if status == "InstallRebooting":
            # 桩正在重启安装固件 → 等待 BootNotification
            self.event_bus.emit("firmware_rebooting", pile_id=pile_id)
        elif status in ("DownloadFailed", "InstallFailed"):
            # 固件更新失败 → 记录并告警
            self.event_bus.emit("firmware_failed", pile_id=pile_id, status=status)
        elif status == "Downloaded":
            # 下载完成，触发安装
            self._trigger_firmware_install(pile_id)

        return {}

    # ========== 下行消息发送：云端 → 充电桩 ==========

    async def send_remote_start(self, pile_id: str, user_id: str,
                                 connector_id: int = 1,
                                 charging_profile: Optional[ChargingProfile] = None) -> dict:
        """远程启动充电"""
        payload = {
            "connectorId": connector_id,
            "idToken": {"idToken": user_id, "type": "Central"},
        }
        if charging_profile:
            payload["chargingProfile"] = self._serialize_profile(charging_profile)

        response = await self._send_call(pile_id, OCPPMessageAction.REMOTE_START, payload)
        return response

    async def send_remote_stop(self, pile_id: str, transaction_id: str) -> dict:
        """远程停止充电"""
        payload = {"transactionId": transaction_id}
        response = await self._send_call(pile_id, OCPPMessageAction.REMOTE_STOP, payload)
        return response

    async def send_set_power_limit(self, pile_id: str, power_watts: int,
                                    transaction_id: Optional[str] = None) -> dict:
        """下发功率限制"""
        profile = self._build_power_limit_profile(transaction_id, power_watts)
        payload = {
            "connectorId": 0,   # 0 = 整桩
            "chargingProfile": self._serialize_profile(profile),
        }
        response = await self._send_call(pile_id, OCPPMessageAction.SET_VARIABLE, payload)
        return response

    async def send_reset(self, pile_id: str, reset_type: str = "Soft") -> dict:
        """下发重启指令（Soft=软重启 / Hard=断电重启）"""
        payload = {"type": reset_type}
        response = await self._send_call(pile_id, OCPPMessageAction.RESET, payload)
        return response

    async def send_unlock_connector(self, pile_id: str, connector_id: int = 1) -> dict:
        """远程解锁充电枪（紧急释放）"""
        payload = {"connectorId": connector_id}
        response = await self._send_call(pile_id, OCPPMessageAction.UNLOCK_CONNECTOR, payload)
        return response

    # ========== 辅助方法 ==========

    def _build_power_limit_profile(self, transaction_id: Optional[str],
                                    max_power_watts: int) -> ChargingProfile:
        """构建功率限制充电策略"""
        return ChargingProfile(
            profile_id=int(datetime.utcnow().timestamp()),
            stack_level=10,   # 高优先级，覆盖默认策略
            charging_profile_purpose="TxProfile" if transaction_id else "ChargingStationMaxProfile",
            charging_profile_kind=ChargingProfileKind.ABSOLUTE,
            schedule=ChargingSchedule(
                duration=86400,  # 24小时
                charging_rate_unit="W",
                periods=[
                    ChargingSchedulePeriod(start_period=0, limit=max_power_watts)
                ],
                start_schedule=datetime.utcnow(),
            ),
            transaction_id=transaction_id,
        )

    def _serialize_profile(self, profile: ChargingProfile) -> dict:
        """序列化充电策略为OCPP JSON格式"""
        result = {
            "chargingProfileId": profile.profile_id,
            "stackLevel": profile.stack_level,
            "chargingProfilePurpose": profile.charging_profile_purpose,
            "chargingProfileKind": profile.charging_profile_kind.value,
            "chargingSchedule": {
                "duration": profile.schedule.duration,
                "chargingRateUnit": profile.schedule.charging_rate_unit,
                "chargingSchedulePeriod": [
                    {
                        "startPeriod": p.start_period,
                        "limit": p.limit,
                        "numberPhases": p.number_phases,
                    }
                    for p in profile.schedule.periods
                ],
                "minChargingRate": profile.schedule.min_charging_rate,
            }
        }
        if profile.start_schedule:
            result["chargingSchedule"]["startSchedule"] = profile.start_schedule.isoformat()
        if profile.transaction_id:
            result["transactionId"] = profile.transaction_id
        return result

    async def _send_call(self, pile_id: str, action: OCPPMessageAction,
                          payload: dict) -> dict:
        """发送OCPP CALL消息并等待响应"""
        request_id = str(uuid.uuid4())
        message = [2, request_id, action.value, payload]  # OCPP CALL格式

        # 通过WebSocket发送（OCPP 2.0.1 使用WebSocket传输）
        if pile_id not in self.charge_points:
            logger.warning(f"Pile {pile_id} not connected, cannot send {action.value}")
            return {"status": "Rejected", "statusInfo": {"reasonCode": "NotConnected"}}

        ws = self.charge_points[pile_id]["websocket"]
        await ws.send(json.dumps(message))

        # 设置超时等待响应
        try:
            response = await self._wait_for_response(request_id, timeout=10)
            return response
        except TimeoutError:
            logger.error(f"OCPP CALL timeout: {action.value} to {pile_id}")
            return {"status": "Rejected", "statusInfo": {"reasonCode": "Timeout"}}

    async def _wait_for_response(self, request_id: str, timeout: int = 10) -> dict:
        """等待OCPP CALL响应"""
        import asyncio
        future = asyncio.get_event_loop().create_future()
        self.pending_requests[request_id] = future
        try:
            return await asyncio.wait_for(future, timeout=timeout)
        finally:
            self.pending_requests.pop(request_id, None)

    def _handle_high_soc(self, pile_id: str, transaction_id: str, soc: float):
        """高SOC时的智能调度——锂电80%以上充电效率急剧下降，可降功率释放给其他用户"""
        if soc >= 95:
            # SOC ≥ 95%，大幅降功率（涓流充电阶段）
            self.power_scheduler.derate_pile(pile_id, target_power_kw=7)
        elif soc >= 85:
            # SOC 85-95%，中等降功率
            pile_info = self.station_repo.get_pile(pile_id)
            if pile_info["pile_type"] == "fast":
                self.power_scheduler.derate_pile(pile_id, target_power_kw=30)

    def _handle_over_temperature(self, pile_id: str, temperature: float):
        """过温保护处理"""
        if temperature >= 95:
            # 严重过温：紧急停止
            logger.critical(f"Over-temperature at {pile_id}: {temperature}°C, emergency stop!")
            self.event_bus.emit("emergency_stop", pile_id=pile_id,
                               reason=f"over_temperature:{temperature}")
        elif temperature >= 85:
            # 过温预警：降功率50%
            self.power_scheduler.derate_pile(pile_id, factor=0.5)
            self.event_bus.emit("temperature_warning", pile_id=pile_id,
                               temperature=temperature)

    async def _emergency_stop(self, pile_id: str, reason: str):
        """紧急停止充电"""
        # 立即下发停止指令
        order = self.order_repo.get_active_order_by_pile(pile_id)
        if order:
            await self.send_remote_stop(pile_id, order["order_id"])

        # 更新桩状态为故障
        self.station_repo.update_pile(pile_id, {"status": "fault"})
        # 释放功率预算
        pile_info = self.station_repo.get_pile(pile_id)
        self.power_scheduler.release_power(pile_info["station_id"], pile_id)
        # 通知运维
        self.event_bus.emit("emergency_stop_executed",
                           pile_id=pile_id, reason=reason)

    def _trigger_firmware_install(self, pile_id: str):
        """触发固件安装"""
        # 通过OCPP TriggerMessage请求桩上报安装状态
        self.event_bus.emit("trigger_firmware_install", pile_id=pile_id)

    # ========== WebSocket 连接管理 ==========

    async def on_websocket_connect(self, pile_id: str, websocket):
        """充电桩WebSocket连接建立"""
        self.charge_points[pile_id] = {
            "websocket": websocket,
            "connected_at": datetime.utcnow(),
            "last_message_at": datetime.utcnow(),
        }
        logger.info(f"Charge point {pile_id} connected via WebSocket")

    async def on_websocket_disconnect(self, pile_id: str, reason: str = ""):
        """充电桩WebSocket断开"""
        self.charge_points.pop(pile_id, None)
        logger.warning(f"Charge point {pile_id} disconnected: {reason}")

        # 如果桩正在充电，标记为离线
        pile_info = self.station_repo.get_pile(pile_id)
        if pile_info and pile_info["status"] in ("charging", "derated"):
            self.station_repo.update_pile(pile_id, {"status": "offline"})
            self.event_bus.emit("pile_disconnected", pile_id=pile_id)

    async def on_websocket_message(self, pile_id: str, raw_message: str):
        """处理收到的OCPP消息"""
        try:
            message = json.loads(raw_message)
            message_type = message[0]

            if message_type == 2:
                # CALL：桩发来的请求
                request_id = message[1]
                action = message[2]
                payload = message[3]

                handler = self._get_handler(action)
                if handler:
                    response_payload = await handler(pile_id, payload)
                    # 发送CALL_RESULT
                    result = [3, request_id, response_payload]
                    ws = self.charge_points.get(pile_id, {}).get("websocket")
                    if ws:
                        await ws.send(json.dumps(result))

            elif message_type == 3:
                # CALL_RESULT：桩的响应
                request_id = message[1]
                payload = message[2]
                future = self.pending_requests.get(request_id)
                if future and not future.done():
                    future.set_result(payload)

            elif message_type == 4:
                # CALL_ERROR：桩返回错误
                request_id = message[1]
                error_code = message[2]
                error_desc = message[3]
                logger.error(f"OCPP error from {pile_id}: {error_code} - {error_desc}")
                future = self.pending_requests.get(request_id)
                if future and not future.done():
                    future.set_exception(Exception(f"OCPP error: {error_code}"))

        except json.JSONDecodeError:
            logger.error(f"Invalid JSON from {pile_id}: {raw_message[:200]}")
        except Exception as e:
            logger.exception(f"Error processing message from {pile_id}: {e}")

    def _get_handler(self, action: str):
        """根据OCPP动作获取对应的处理方法"""
        handler_map = {
            "BootNotification": self.handle_boot_notification,
            "Heartbeat": self.handle_heartbeat,
            "Authorize": self.handle_authorize,
            "StartTransaction": self.handle_start_transaction,
            "StopTransaction": self.handle_stop_transaction,
            "MeterValues": self.handle_meter_values,
            "StatusNotification": self.handle_status_notification,
            "AlarmSignal": self.handle_alarm_signal,
            "FirmwareStatusNotification": self.handle_firmware_status,
        }
        return handler_map.get(action)
```

**OCPP 通信时序：**

```
充电桩上电流程：
  桩 ──BootNotification──→ CSMS
  桩 ←─Accepted(interval=30)── CSMS
  桩 ──StatusNotification(Available)──→ CSMS

用户扫码充电流程：
  桩 ──Authorize(idToken)──→ CSMS
  桩 ←─Accepted── CSMS
  桩 ──StartTransaction──→ CSMS
  桩 ←─Accepted(transactionId, chargingProfile)── CSMS
  桩 ──MeterValues(periodic, every 60s)──→ CSMS
  ...
  桩 ──StopTransaction──→ CSMS
  桩 ←─Accepted── CSMS

远程功率调整流程：
  CSMS ──SetChargingProfile(powerLimit)──→ 桩
  CSMS ←─Accepted── 桩

紧急停止流程：
  CSMS ──RemoteStopTransaction──→ 桩
  CSMS ←─Accepted── 桩
  桩 ──StopTransaction(reason=RemoteStop)──→ CSMS
```

### 离线检测与自动恢复

```python
class PileHealthMonitor:
    """充电桩健康监控：检测离线和故障"""

    CHECK_INTERVAL = 60  # 每60秒检查一次

    def check_all_piles(self):
        """扫描所有桩的心跳"""
        piles = self.db.query("SELECT pile_id, station_id, status, last_heartbeat FROM charging_piles")

        for pile in piles:
            last_hb = pile["last_heartbeat"]
            offline_threshold = 60  # 60秒无心跳视为离线

            if pile["status"] != "offline" and \
               (now() - last_hb).total_seconds() > offline_threshold:
                self.mark_offline(pile["pile_id"], pile["station_id"])

    def mark_offline(self, pile_id, station_id):
        """标记桩离线并处理影响"""
        # 1. 更新状态
        self.db.update("charging_piles",
                       {"status": "offline"},
                       {"pile_id": pile_id})

        # 2. 如果桩正在充电 → 通知用户换桩
        order = self.db.query_one(
            "SELECT order_id, user_id FROM charge_orders "
            "WHERE pile_id = %s AND charge_status = 'charging'", pile_id)
        if order:
            self.notify_user(order["user_id"],
                             f"充电桩{pile_id}异常，请移至附近可用桩继续充电")
            # 自动分配附近可用桩
            nearby_pile = self.find_available_pile(station_id, pile_id)
            if nearby_pile:
                self.auto_switch(order["order_id"], pile_id, nearby_pile)

        # 3. 释放功率预算
        self.power_scheduler.release_power(station_id, pile_id)

        # 4. 触发排队唤醒
        self.power_scheduler.process_wait_queue(station_id)

    def find_available_pile(self, station_id, exclude_pile_id):
        """在站内寻找可用备用桩"""
        available = self.db.query(
            "SELECT pile_id, pile_type, max_power_kw FROM charging_piles "
            "WHERE station_id = %s AND status = 'idle "
            "AND pile_id != %s "
            "ORDER BY max_power_kw DESC LIMIT 1",  # 优先快充桩
            station_id, exclude_pile_id
        )
        return available[0] if available else None

    def auto_switch(self, order_id, old_pile_id, new_pile_id):
        """自动切换到备用桩"""
        # 更新订单关联的桩
        self.db.update("charge_orders",
                       {"pile_id": new_pile_id},
                       {"order_id": order_id})
        # 下发充电指令到新桩
        self.mqtt_protocol.send_start_charge(new_pile_id, 60)  # 默认满功率
```

### 功率调度：动态分配 + 降功率优先

#### 基础调度器

```python
class PowerScheduler:
    """站点级功率调度器"""

    def __init__(self, station_id, max_power_kw):
        self.station_id = station_id
        self.max_power_kw = max_power_kw
        self.charging_piles = {}  # pile_id → {power_kw, priority, start_time}
        self.wait_queue = []      # 排队队列

    def request_charge(self, pile_id, requested_power_kw, priority="normal"):
        """请求充电"""
        current_total = sum(p["power_kw"] for p in self.charging_piles.values())

        # 策略1：功率充足 → 直接满功率充电
        if current_total + requested_power_kw <= self.max_power_kw:
            self.charging_piles[pile_id] = {
                "power_kw": requested_power_kw,
                "priority": priority,
                "start_time": now()
            }
            return {"status": "approved", "power_kw": requested_power_kw,
                    "estimated_duration": self.estimate_duration(requested_power_kw)}

        # 策略2：功率不足但有剩余 → 降功率充电
        available = self.max_power_kw - current_total
        if available > 7:  # 至少 7kW（慢充功率）
            actual_power = min(requested_power_kw, available)
            self.charging_piles[pile_id] = {
                "power_kw": actual_power,
                "priority": priority,
                "start_time": now()
            }
            # 通知用户：充电速度降低
            duration = self.estimate_duration(actual_power)
            return {"status": "derated", "power_kw": actual_power,
                    "estimated_duration": duration,
                    "note": f"当前功率{actual_power}kW，预计充电{duration}分钟"}

        # 策略3：完全无可用功率 → 排队
        self.wait_queue.append({
            "pile_id": pile_id,
            "requested_power": requested_power_kw,
            "priority": priority,
            "queued_at": now()
        })
        position = len(self.wait_queue)
        estimated_wait = self.estimate_wait_time()
        return {"status": "queued", "position": position,
                "estimated_wait": estimated_wait}

    def estimate_duration(self, power_kw):
        """根据充电功率估算充电时间"""
        # 快充 60kW → 约 40 分钟充 80%
        # 降功率 30kW → 约 80 分钟
        # 慢充 7kW → 约 6 小时
        if power_kw >= 60:
            return 40
        elif power_kw >= 30:
            return 80
        else:
            return 360  # 6小时

    def estimate_wait_time(self):
        """估算排队等待时间"""
        if not self.wait_queue:
            return 0

        # 基于当前在充车辆的预计完成时间
        finish_times = [p["start_time"] + timedelta(minutes=self.estimate_duration(p["power_kw"]))
                       for p in self.charging_piles.values()]

        earliest_finish = min(finish_times)
        wait = (earliest_finish - now()).total_seconds() / 60

        return max(0, round(wait))

    def on_charge_complete(self, pile_id):
        """充电完成 → 释放功率 → 唤醒排队用户"""
        power_released = self.charging_piles.get(pile_id, {}).get("power_kw", 0)
        if pile_id in self.charging_piles:
            del self.charging_piles[pile_id]

        # 记录调度日志
        self.log_schedule_action(
            action="release", pile_id=pile_id,
            actual_power_kw=0, released_power=power_released
        )

        # 尝试升级降功率车辆
        self.upgrade_all_derated()

        # 唤醒排队用户
        if self.wait_queue:
            next_in_queue = self.wait_queue.pop(0)
            result = self.request_charge(
                next_in_queue["pile_id"],
                next_in_queue["requested_power"],
                next_in_queue["priority"]
            )
            self.notify_user(next_in_queue.get("user_id"),
                           f"充电桩已就绪，预计{result.get('estimated_duration', '?')}分钟完成")

    def on_vehicle_left(self, pile_id):
        """车辆离开 → 释放桩位和功率"""
        if pile_id in self.charging_piles:
            del self.charging_piles[pile_id]
        # 不同于充电完成：车辆离开只是释放占位，功率已在充电完成时释放
        self.process_wait_queue()
```

**降功率 vs 排队的权衡：**

| 策略 | 用户等待 | 充电速度 | 体验 |
|------|---------|---------|------|
| 降功率30kW | 0（立即开始） | 慢（80分钟） | 可接受 |
| 排队等60kW | 30-40分钟 | 快（40分钟） | 总时间更长 |
| 混合（先降功率，有空位时升级） | 0→升级 | 渐快 | 最佳 |

**推荐策略：先降功率充电，功率释放后自动升级到满功率。**

```python
    def upgrade_all_derated(self):
        """功率释放后，尝试升级所有降功率车辆"""
        current_total = sum(p["power_kw"] for p in self.charging_piles.values())

        # 按降功率程度排序：被降得最多的优先升级
        derated = [
            (pid, p) for pid, p in self.charging_piles.items()
            if p["power_kw"] < p.get("original_power_kw", p["power_kw"])
        ]
        derated.sort(key=lambda x: x[1]["power_kw"])  # 功率最低的优先

        for pile_id, info in derated:
            target_power = info.get("original_power_kw", 60)
            upgrade_needed = target_power - info["power_kw"]

            if current_total + upgrade_needed <= self.max_power_kw:
                info["power_kw"] = target_power
                current_total += upgrade_needed
                self.mqtt_protocol.send_set_power(pile_id, target_power)
                self.notify_user(info.get("user_id"),
                               f"充电速度已升级到{target_power}kW")

    def upgrade_power(self, pile_id):
        """单个桩的功率升级"""
        if pile_id not in self.charging_piles:
            return

        current = self.charging_piles[pile_id]
        target_power = current.get("original_power_kw", 60)
        if current["power_kw"] < target_power:
            total_with_upgrade = sum(p["power_kw"] for p in self.charging_piles.values()) \
                                - current["power_kw"] + target_power

            if total_with_upgrade <= self.max_power_kw:
                current["power_kw"] = target_power
                self.mqtt_protocol.send_set_power(pile_id, target_power)
                self.notify_user(current.get("user_id"),
                               f"充电速度已升级到{target_power}kW")
```

#### 高级约束满足调度算法

基础调度器采用贪心策略，在功率约束简单时可行。但实际场景中，站点可能有多种约束叠加——总功率上限、变压器容量、电缆载流量、谷时优先等。需要一个基于约束满足的优化调度器，在多约束条件下求解最优功率分配方案。

```python
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
from enum import Enum
import math
import logging

logger = logging.getLogger(__name__)


class ChargingPriority(Enum):
    """充电优先级"""
    EMERGENCY = 100     # 紧急充电（续航<10%的车辆）
    HIGH = 80           # 高优先级（预约用户、VIP）
    NORMAL = 50         # 普通优先级
    LOW = 20            # 低优先级（可延后充电、谷时预约）
    OPPORTUNISTIC = 10  # 随机充电（谷时闲充）


@dataclass
class ChargingSession:
    """充电会话——调度算法的输入单元"""
    session_id: str
    pile_id: str
    station_id: str
    user_id: str
    priority: ChargingPriority
    requested_power_kw: float          # 用户期望功率
    min_power_kw: float                # 最低可接受功率（低于此则排队）
    original_power_kw: float           # 桩的最大功率
    current_power_kw: float = 0.0      # 当前实际分配功率
    current_soc: float = 0.0           # 电池当前SOC（%）
    target_soc: float = 80.0           # 目标SOC
    battery_capacity_kwh: float = 60.0 # 电池容量
    start_time: Optional[datetime] = None
    estimated_departure: Optional[datetime] = None
    is_derated: bool = False           # 是否处于降功率状态

    @property
    def energy_needed_kwh(self) -> float:
        """还需充入的电量"""
        return max(0, self.battery_capacity_kwh * (self.target_soc - self.current_soc) / 100)

    @property
    def estimated_remaining_minutes(self) -> float:
        """按当前功率估算剩余时间"""
        if self.current_power_kw <= 0:
            return float('inf')
        # 考虑锂电充电曲线：SOC 0-80%恒流，80-100%涓流
        effective_power = self.current_power_kw
        if self.current_soc > 80:
            effective_power *= (100 - self.current_soc) / 40  # 涓流衰减
        return (self.energy_needed_kwh / effective_power) * 60

    @property
    def urgency_score(self) -> float:
        """紧急度评分（0-1，越高越紧急）"""
        if not self.estimated_departure:
            return 0.3
        time_until_departure = (self.estimated_departure - datetime.utcnow()).total_seconds() / 60
        if time_until_departure <= 0:
            return 1.0
        # 剩余时间越短、能量需求越多 → 越紧急
        time_ratio = max(0, 1 - time_until_departure / 120)  # 2小时内线性增加
        energy_ratio = min(1, self.energy_needed_kwh / 40)    # 需要充40度以上=最紧急
        return min(1.0, time_ratio * 0.6 + energy_ratio * 0.4)


@dataclass
class StationConstraint:
    """站点约束条件"""
    station_id: str
    max_total_power_kw: float           # 变压器容量上限
    max_fast_charge_power_kw: float     # 快充区域功率上限
    max_slow_charge_power_kw: float     # 慢充区域功率上限
    auxiliary_power_kw: float = 0.0     # 辅助设备耗电（空调、照明等）
    cable_current_limit_a: float = 400  # 进线电缆载流量（安培）
    voltage: float = 380.0              # 线电压

    @property
    def available_power_kw(self) -> float:
        """可用功率 = 总容量 - 辅助耗电 - 安全裕量"""
        safety_margin = 0.05  # 5%安全裕量，防止瞬态过载
        return self.max_total_power_kw * (1 - safety_margin) - self.auxiliary_power_kw

    @property
    def cable_limit_kw(self) -> float:
        """电缆载流功率上限（三相）"""
        return math.sqrt(3) * self.voltage * self.cable_current_limit_a / 1000


class ConstraintSatisfactionScheduler:
    """
    基于约束满足的功率调度器

    目标函数：最大化总充电效用（utility）
    约束条件：
      1. 站点总功率 ≤ 可用功率
      2. 快充区域功率 ≤ 快充上限
      3. 慢充区域功率 ≤ 慢充上限
      4. 电缆载流 ≤ 电缆上限
      5. 单桩功率 ≥ 最低可接受功率（否则排队）
      6. 单桩功率 ≤ 桩最大功率

    效用函数综合考虑：用户优先级 × 紧急度 × 功率边际效用
    """

    def __init__(self, redis_client, mqtt_client, db_client):
        self.redis = redis_client
        self.mqtt = mqtt_client
        self.db = db_client
        # 每个站点一个调度实例的状态缓存
        self._station_states: Dict[str, dict] = {}

    def schedule_station(self, station_id: str,
                          sessions: List[ChargingSession],
                          constraint: StationConstraint) -> Dict[str, dict]:
        """
        对一个充电站执行完整的功率调度

        返回：{ session_id: { "power_kw": float, "action": str, "message": str } }
        """
        # 步骤1：计算每个会话的效用权重
        weighted_sessions = self._calculate_weights(sessions)

        # 步骤2：约束传播——预先排除不可能满足的会话
        active_sessions, queued_sessions = self._constraint_propagation(
            weighted_sessions, constraint)

        # 步骤3：求解最优功率分配
        allocation = self._solve_optimal_allocation(active_sessions, constraint)

        # 步骤4：后处理——验证约束、生成指令、通知用户
        result = self._post_process(allocation, queued_sessions)

        # 步骤5：持久化调度结果
        self._persist_schedule(station_id, allocation, queued_sessions)

        return result

    def _calculate_weights(self, sessions: List[ChargingSession]
                           ) -> List[Tuple[float, ChargingSession]]:
        """
        计算每个充电会话的综合权重

        权重 = 优先级分 × 紧急度 × SOC惩罚因子
        SOC惩罚：低SOC的车辆权重更高（救急优先）
        """
        weighted = []
        for s in sessions:
            # SOC惩罚因子：SOC越低权重越高
            soc_factor = max(0.5, (100 - s.current_soc) / 100)

            # 综合权重
            weight = (s.priority.value / 100) * s.urgency_score * soc_factor

            # 功率边际效用递减：从0kW到30kW效用高，30kW到60kW效用递减
            # 这意味着降功率时，先降高功率桩的效果更好
            power_utility = math.log2(max(1, s.current_power_kw)) if s.current_power_kw > 0 else 0

            combined_weight = weight + power_utility * 0.1
            weighted.append((combined_weight, s))

        # 按权重降序排列
        weighted.sort(key=lambda x: x[0], reverse=True)
        return weighted

    def _constraint_propagation(self,
                                 weighted_sessions: List[Tuple[float, ChargingSession]],
                                 constraint: StationConstraint
                                 ) -> Tuple[List[Tuple[float, ChargingSession]],
                                            List[ChargingSession]]:
        """
        约束传播：预先判断哪些会话不可能被满足，直接进入排队

        判断逻辑：
        - 如果一个会话的最低功率 + 当前已分配功率 > 可用功率，且没有可降功率的空间 → 排队
        """
        active = []
        queued = []

        current_power = sum(s.current_power_kw for _, s in weighted_sessions)
        available = constraint.available_power_kw

        for weight, session in weighted_sessions:
            if current_power + session.min_power_kw <= available:
                active.append((weight, session))
                current_power += session.current_power_kw  # 保留当前功率
            else:
                # 尝试通过降功率已有会话来腾出空间
                freed = self._try_free_power(active, session.min_power_kw,
                                              available - current_power)
                if freed >= session.min_power_kw:
                    current_power -= freed - session.min_power_kw
                    active.append((weight, session))
                else:
                    queued.append(session)

        return active, queued

    def _try_free_power(self, active_sessions: List[Tuple[float, ChargingSession]],
                         needed_kw: float, currently_available: float) -> float:
        """
        尝试从已有活跃会话中释放功率

        策略：从权重最低的会话开始降功率，直到释放足够功率
        返回：实际释放的功率
        """
        if currently_available >= needed_kw:
            return needed_kw

        deficit = needed_kw - currently_available
        freed = 0.0

        # 从权重最低的会话开始降功率
        sorted_sessions = sorted(active_sessions, key=lambda x: x[0])

        for weight, session in sorted_sessions:
            if freed >= deficit:
                break

            # 计算可降功率空间
            reducible = session.current_power_kw - session.min_power_kw
            if reducible > 0:
                # 降功率到最低可接受功率
                actual_reduce = min(reducible, deficit - freed)
                freed += actual_reduce
                session.current_power_kw -= actual_reduce
                session.is_derated = True

        return currently_available + freed

    def _solve_optimal_allocation(self,
                                   active_sessions: List[Tuple[float, ChargingSession]],
                                   constraint: StationConstraint
                                   ) -> Dict[str, float]:
        """
        求解最优功率分配

        采用拉格朗日松弛法 + 贪心分配：
        1. 先按权重分配满功率
        2. 如果超约束，从最低权重开始降功率
        3. 降功率时采用等比例缩减（公平性）
        4. 迭代直到满足所有约束
        """
        allocation = {}
        total_power = 0.0

        # 第一轮：按权重从高到低分配满功率
        for weight, session in active_sessions:
            power = min(session.requested_power_kw, session.original_power_kw)
            allocation[session.session_id] = power
            total_power += power

        # 第二轮：约束检查与功率调整
        max_iterations = 20
        iteration = 0

        while iteration < max_iterations:
            iteration += 1
            violations = self._check_constraints(allocation, active_sessions, constraint)

            if not violations:
                break  # 所有约束满足

            for violation in violations:
                if violation["type"] == "total_power":
                    # 总功率超标 → 等比例缩减降功率会话的功率
                    excess = violation["excess"]
                    self._proportional_reduction(allocation, active_sessions, excess)

                elif violation["type"] == "cable_limit":
                    # 电缆载流超标 → 优先降低高功率桩
                    excess = violation["excess"]
                    self._reduce_high_power_first(allocation, active_sessions, excess)

        # 第三轮：碎片功率回收——检查是否有零散的可用功率可以分配
        self._reclaim_fragmented_power(allocation, active_sessions, constraint)

        return allocation

    def _check_constraints(self, allocation: Dict[str, float],
                            active_sessions: List[Tuple[float, ChargingSession]],
                            constraint: StationConstraint
                            ) -> List[dict]:
        """检查所有约束，返回违规列表"""
        violations = []

        # 约束1：总功率上限
        total = sum(allocation.values())
        if total > constraint.available_power_kw:
            violations.append({
                "type": "total_power",
                "excess": total - constraint.available_power_kw,
                "current": total,
                "limit": constraint.available_power_kw,
            })

        # 约束2：电缆载流上限
        if total > constraint.cable_limit_kw:
            violations.append({
                "type": "cable_limit",
                "excess": total - constraint.cable_limit_kw,
                "current": total,
                "limit": constraint.cable_limit_kw,
            })

        # 约束3：快充区域功率上限
        fast_total = sum(allocation.get(s.session_id, 0)
                        for _, s in active_sessions
                        if s.original_power_kw >= 30)
        if fast_total > constraint.max_fast_charge_power_kw:
            violations.append({
                "type": "fast_charge_area",
                "excess": fast_total - constraint.max_fast_charge_power_kw,
                "current": fast_total,
                "limit": constraint.max_fast_charge_power_kw,
            })

        # 约束4：慢充区域功率上限
        slow_total = sum(allocation.get(s.session_id, 0)
                        for _, s in active_sessions
                        if s.original_power_kw < 30)
        if slow_total > constraint.max_slow_charge_power_kw:
            violations.append({
                "type": "slow_charge_area",
                "excess": slow_total - constraint.max_slow_charge_power_kw,
                "current": slow_total,
                "limit": constraint.max_slow_charge_power_kw,
            })

        # 约束5：单桩功率范围
        session_map = {s.session_id: s for _, s in active_sessions}
        for sid, power in allocation.items():
            session = session_map.get(sid)
            if session:
                if power < session.min_power_kw:
                    violations.append({
                        "type": "below_min",
                        "session_id": sid,
                        "current": power,
                        "min": session.min_power_kw,
                    })
                if power > session.original_power_kw:
                    violations.append({
                        "type": "above_max",
                        "session_id": sid,
                        "current": power,
                        "max": session.original_power_kw,
                    })

        return violations

    def _proportional_reduction(self, allocation: Dict[str, float],
                                 active_sessions: List[Tuple[float, ChargingSession]],
                                 excess_kw: float):
        """
        等比例缩减功率

        策略：只缩减可缩减的会话（功率 > 最低可接受功率）
        缩减比例 = excess / total_reducible
        """
        session_map = {s.session_id: s for _, s in active_sessions}

        # 计算可缩减总量
        reducible_sessions = []
        total_reducible = 0.0
        for sid, power in allocation.items():
            session = session_map.get(sid)
            if session and power > session.min_power_kw:
                reducible = power - session.min_power_kw
                reducible_sessions.append((sid, power, session, reducible))
                total_reducible += reducible

        if total_reducible <= 0:
            return  # 无法缩减

        # 按权重从低到高排序，低权重的多缩减
        reducible_sessions.sort(key=lambda x: x[2].priority.value)

        # 分配缩减量
        remaining_excess = excess_kw
        for sid, current_power, session, reducible in reducible_sessions:
            if remaining_excess <= 0:
                break

            # 该会话应缩减的量
            share = min(reducible, remaining_excess)
            new_power = current_power - share
            new_power = max(session.min_power_kw, new_power)

            allocation[sid] = new_power
            remaining_excess -= (current_power - new_power)

    def _reduce_high_power_first(self, allocation: Dict[str, float],
                                  active_sessions: List[Tuple[float, ChargingSession]],
                                  excess_kw: float):
        """优先降低高功率桩的功率（电缆约束场景）"""
        session_map = {s.session_id: s for _, s in active_sessions}

        # 按当前功率从高到低排序
        sorted_alloc = sorted(allocation.items(), key=lambda x: x[1], reverse=True)

        remaining_excess = excess_kw
        for sid, power in sorted_alloc:
            if remaining_excess <= 0:
                break
            session = session_map.get(sid)
            if session and power > session.min_power_kw:
                reduction = min(power - session.min_power_kw, remaining_excess)
                allocation[sid] = power - reduction
                remaining_excess -= reduction

    def _reclaim_fragmented_power(self, allocation: Dict[str, float],
                                   active_sessions: List[Tuple[float, ChargingSession]],
                                   constraint: StationConstraint):
        """回收碎片功率——降功率会话释放的空间可能可以部分利用"""
        session_map = {s.session_id: s for _, s in active_sessions}

        total_allocated = sum(allocation.values())
        available = constraint.available_power_kw - total_allocated

        if available <= 0:
            return

        # 按权重从高到低，检查是否可以升级降功率会话
        sorted_sessions = sorted(active_sessions, key=lambda x: x[0], reverse=True)

        for weight, session in sorted_sessions:
            if available <= 0:
                break

            current_power = allocation.get(session.session_id, 0)
            if current_power < session.original_power_kw:
                upgrade = min(session.original_power_kw - current_power, available)
                allocation[session.session_id] = current_power + upgrade
                available -= upgrade

    def _post_process(self, allocation: Dict[str, float],
                       queued_sessions: List[ChargingSession]
                       ) -> Dict[str, dict]:
        """后处理：生成指令和通知"""
        result = {}

        for session_id, power_kw in allocation.items():
            action = "maintain"
            message = ""

            # 确定动作类型
            session = self._get_session(session_id)
            if not session:
                continue

            if power_kw != session.current_power_kw:
                if power_kw < session.current_power_kw:
                    action = "derate"
                    message = f"功率调整为{power_kw:.0f}kW，预计充电{self._estimate_duration(power_kw)}分钟"
                elif power_kw > session.current_power_kw:
                    action = "upgrade"
                    message = f"功率升级为{power_kw:.0f}kW，充电速度提升"
                else:
                    action = "maintain"

            result[session_id] = {
                "power_kw": power_kw,
                "action": action,
                "message": message,
                "pile_id": session.pile_id,
                "is_derated": power_kw < session.original_power_kw,
            }

        for session in queued_sessions:
            result[session.session_id] = {
                "power_kw": 0,
                "action": "queue",
                "message": f"当前功率不足，请排队等待，预计{self._estimate_queue_wait(session)}分钟",
                "pile_id": session.pile_id,
                "is_derated": False,
            }

        return result

    def _estimate_duration(self, power_kw: float) -> int:
        """估算充电时间（分钟）"""
        if power_kw >= 60:
            return 40
        elif power_kw >= 40:
            return 55
        elif power_kw >= 30:
            return 80
        elif power_kw >= 15:
            return 160
        else:
            return 360

    def _estimate_queue_wait(self, session: ChargingSession) -> int:
        """估算排队等待时间"""
        # 基于当前充电会话的预计完成时间
        station_state = self._station_states.get(session.station_id, {})
        active_sessions = station_state.get("active_sessions", [])

        if not active_sessions:
            return 5  # 默认5分钟

        # 计算最早释放足够功率的时间
        finish_times = []
        for s in active_sessions:
            remaining = s.estimated_remaining_minutes
            finish_times.append(remaining)

        finish_times.sort()
        # 需要等待的会话数量 = 当前排队位置
        position = len(station_state.get("wait_queue", []))
        if position < len(finish_times):
            return max(5, int(finish_times[position]))
        return max(5, int(finish_times[-1]) if finish_times else 30)

    def _persist_schedule(self, station_id: str,
                           allocation: Dict[str, float],
                           queued_sessions: List[ChargingSession]):
        """持久化调度结果到数据库（审计用）"""
        for session_id, power_kw in allocation.items():
            self.db.insert("power_schedule_log", {
                "station_id": station_id,
                "pile_id": session_id,  # session_id映射到pile_id
                "action": "allocate",
                "actual_power_kw": power_kw,
                "created_at": datetime.utcnow(),
            })

        for session in queued_sessions:
            self.db.insert("power_schedule_log", {
                "station_id": station_id,
                "pile_id": session.pile_id,
                "action": "queue",
                "requested_power_kw": session.requested_power_kw,
                "actual_power_kw": 0,
                "created_at": datetime.utcnow(),
            })

        # 更新Redis缓存
        self.redis.hset(
            f"station:{station_id}:schedule",
            mapping={sid: str(p) for sid, p in allocation.items()}
        )

    def _get_session(self, session_id: str) -> Optional[ChargingSession]:
        """从缓存获取会话信息"""
        for station_state in self._station_states.values():
            for s in station_state.get("active_sessions", []):
                if s.session_id == session_id:
                    return s
        return None

    # ========== 定时调度入口 ==========

    def rebalance_station(self, station_id: str):
        """
        站点功率再平衡——当功率变化时触发

        触发场景：
        1. 新充电请求
        2. 充电完成释放功率
        3. SOC上报导致优先级变化
        4. 辅助设备功率变化（空调启停）
        """
        # 从缓存加载当前状态
        sessions = self._load_active_sessions(station_id)
        constraint = self._load_station_constraint(station_id)

        # 执行调度
        result = self.schedule_station(station_id, sessions, constraint)

        # 下发功率调整指令
        for session_id, info in result.items():
            if info["action"] == "derate":
                self.mqtt.publish(
                    f"cmd/{info['pile_id']}/power",
                    json.dumps({"command": "set_power", "power_kw": info["power_kw"]})
                )
            elif info["action"] == "upgrade":
                self.mqtt.publish(
                    f"cmd/{info['pile_id']}/power",
                    json.dumps({"command": "set_power", "power_kw": info["power_kw"]})
                )
            elif info["action"] == "queue":
                self.mqtt.publish(
                    f"user/{info['pile_id']}/notify",
                    json.dumps({"type": "queue", "message": info["message"]})
                )
```

**调度算法核心设计思路：**

```
输入：N个充电会话 + M个约束条件
输出：每个会话的功率分配方案

Step 1: 权重计算
  weight = f(优先级, 紧急度, SOC, 功率边际效用)
  → 排序得到优先级队列

Step 2: 约束传播（提前剪枝）
  对每个新会话，检查最低功率需求能否满足
  不能满足 → 直接排队，避免浪费求解时间

Step 3: 最优分配求解
  3a. 贪心初始化：按权重分配满功率
  3b. 约束检查：检测所有违规
  3c. 迭代调整：
      - 总功率超标 → 等比例缩减低权重会话
      - 电缆超标 → 优先降低高功率桩
      - 区域超标 → 只缩减该区域会话
  3d. 碎片回收：重新分配释放的空间

Step 4: 指令下发 + 通知用户
```

**调度公平性保障机制：**

| 机制 | 说明 |
|------|------|
| 等比例缩减 | 降功率时按比例分配，避免某个用户被大幅降功率 |
| 最低功率保障 | 每个会话有min_power_kw，低于此值则排队而非降功率 |
| 紧急度加权 | 低SOC车辆权重更高，确保"救急优先" |
| 升级轮转 | 功率释放后，被降功率最久的用户优先升级 |
| 饥饿防止 | 排队超过15分钟自动提升优先级 |

### 预约管理：弹性时间窗 + 占位超时

```python
class ReservationService:
    def reserve(self, user_id, station_id, pile_type, start_time, duration_min):
        """预约充电桩"""
        # 15 分钟宽限：用户迟到 15 分钟内保留预约
        grace_period = 15
        
        # 查找空闲桩
        available_piles = self.find_available(
            station_id, pile_type, start_time,
            start_time + timedelta(minutes=duration_min + grace_period)
        )
        
        if not available_piles:
            return {"status": "no_availability"}
        
        pile = available_piles[0]
        reservation_id = uuid4()
        
        self.db.insert("reservations", {
            "reservation_id": reservation_id,
            "user_id": user_id,
            "pile_id": pile.id,
            "start_time": start_time,
            "end_time": start_time + timedelta(minutes=duration_min),
            "grace_end": start_time + timedelta(minutes=duration_min + grace_period),
            "status": "reserved"
        })
        
        # 设置宽限期到期定时器
        self.schedule_grace_expiry(reservation_id, start_time + timedelta(minutes=grace_period))
        
        return {"status": "reserved", "reservation_id": reservation_id}

    def on_grace_period_expired(self, reservation_id):
        """宽限期过期 → 取消预约 → 释放桩位"""
        reservation = self.db.get("reservations", reservation_id)
        
        if reservation["status"] == "reserved":
            # 用户还没到 → 取消预约
            self.db.update("reservations", {"status": "expired"}, {"reservation_id": reservation_id})
            self.notify_user(reservation["user_id"], "预约已过期，请重新预约")
            # 释放桩位给排队用户
            self.power_scheduler.on_charge_complete(reservation["pile_id"])

    def on_charge_complete_not_left(self, pile_id):
        """充电完成但用户不挪车 → 提醒 + 超时罚则"""
        # 5 分钟提醒
        self.notify_user(user_id, "充电已完成，请在10分钟内挪车")
        
        # 10 分钟超时 → 开始收取占位费（¥0.5/分钟）
        self.schedule_occupancy_fee(pile_id, delay_minutes=10)

    def charge_occupancy_fee(self, pile_id):
        """收取占位费（激励用户挪车）"""
        fee_per_minute = 0.5
        elapsed = (now() - self.get_charge_complete_time(pile_id)).total_seconds() / 60
        
        total_fee = max(0, elapsed - 10) * fee_per_minute  # 10 分钟免费
        
        self.bill_user(user_id, total_fee, reason="占位费")
        self.notify_user(user_id, f"占位费 ¥{total_fee:.1f}，请尽快挪车")
```

### 电价引导：推荐 + 激励

```python
class PricingGuide:
    """分时电价引导"""
    
    TIME_OF_USE = {
        "valley": {"hours": range(22, 24) + range(0, 6), "price_per_kwh": 0.3},
        "flat":   {"hours": range(6, 8) + range(11, 18) + range(21, 22), "price_per_kwh": 0.7},
        "peak":   {"hours": range(8, 11) + range(18, 21), "price_per_kwh": 1.2},
    }

    def get_recommendation(self, user_id, desired_energy_kwh):
        """推荐最优充电时段"""
        options = []
        
        for period, config in self.TIME_OF_USE.items():
            # 充电时间估算（假设快充 60kW）
            hours_needed = desired_energy_kwh / 60
            cost = desired_energy_kwh * config["price_per_kwh"]
            savings_vs_peak = desired_energy_kwh * (1.2 - config["price_per_kwh"])
            
            options.append({
                "period": period,
                "start_hour": config["hours"][0],
                "hours_needed": round(hours_needed, 1),
                "cost": round(cost, 1),
                "savings_vs_peak": round(savings_vs_peak, 1)
            })
        
        options.sort(key=lambda x: x["cost"])
        
        return {
            "recommended": options[0],
            "all_options": options,
            "incentive": self.calculate_incentive(options[0])
        }

    def calculate_incentive(self, best_option):
        """谷时充电激励（折扣/积分）"""
        if best_option["period"] == "valley":
            # 谷时充电额外 10% 折扣
            discount = best_option["cost"] * 0.1
            return {"type": "discount", "amount": round(discount, 1),
                    "message": f"谷时充电额外优惠 ¥{discount:.1f}"}
        return None
```

**电价引导效果估算：**

| 策略 | 谷时利用率提升 | 收入影响 |
|------|--------------|---------|
| 无引导 | 0% | 基线 |
| 推荐+价格展示 | +15% | 略降（谷时单价低） |
| 推荐+折扣激励 | +30% | 谷时收入增加（量增弥补价低） |
| 推荐+占位费 | +20% | 峰时体验改善 |

### 计费与结算系统

充电运营平台的计费比想象中复杂得多——不是简单的"电量 × 单价"。实际需要处理：分时电价、需量电费、服务费分层、占位费、会员折扣、优惠券抵扣、跨时段充电的费率切换等。

#### 分时电价与费率引擎

```python
from dataclasses import dataclass
from typing import List, Optional, Tuple
from datetime import datetime, time, timedelta
from decimal import Decimal, ROUND_HALF_UP
import logging

logger = logging.getLogger(__name__)


@dataclass
class TimeOfUsePeriod:
    """分时电价时段定义"""
    period_id: str
    name: str                   # 尖/峰/平/谷
    start_time: time            # 时段开始
    end_time: time              # 时段结束
    electricity_price: Decimal  # 电费单价（元/kWh）
    service_price: Decimal      # 服务费单价（元/kWh）

    def contains(self, t: time) -> bool:
        """判断时间是否在时段内"""
        if self.start_time <= self.end_time:
            return self.start_time <= t < self.end_time
        else:
            # 跨午夜时段（如谷时 22:00-06:00）
            return t >= self.start_time or t < self.end_time


@dataclass
class DemandChargeConfig:
    """需量电费配置（变压器容量相关）"""
    contract_demand_kw: Decimal       # 合约需量（kW）
    demand_charge_per_kw: Decimal     # 需量电费单价（元/kW/月）
    penalty_threshold: Decimal        # 超限阈值（合约需量的1.05倍）
    penalty_rate: Decimal             # 超限惩罚倍率
    measurement_period_min: int = 15  # 需量测量周期（15分钟）


class BillingRateEngine:
    """
    费率引擎——管理所有计费规则

    支持的计费维度：
    1. 分时电价（尖峰平谷）
    2. 充电桩类型费率差异（快充服务费高于慢充）
    3. 会员等级折扣
    4. 需量电费（月度结算）
    5. 占位费（按分钟计费）
    """

    # 默认分时电价时段（国家电网工商业用电标准）
    DEFAULT_TOU_PERIODS = [
        TimeOfUsePeriod("sharp", "尖", time(10, 0), time(12, 0),
                        Decimal("1.35"), Decimal("0.80")),
        TimeOfUsePeriod("sharp2", "尖", time(19, 0), time(21, 0),
                        Decimal("1.35"), Decimal("0.80")),
        TimeOfUsePeriod("peak1", "峰", time(8, 0), time(10, 0),
                        Decimal("1.20"), Decimal("0.70")),
        TimeOfUsePeriod("peak2", "峰", time(12, 0), time(14, 0),
                        Decimal("1.20"), Decimal("0.70")),
        TimeOfUsePeriod("peak3", "峰", time(17, 0), time(19, 0),
                        Decimal("1.20"), Decimal("0.70")),
        TimeOfUsePeriod("peak4", "峰", time(21, 0), time(22, 0),
                        Decimal("1.20"), Decimal("0.70")),
        TimeOfUsePeriod("flat1", "平", time(6, 0), time(8, 0),
                        Decimal("0.70"), Decimal("0.50")),
        TimeOfUsePeriod("flat2", "平", time(14, 0), time(17, 0),
                        Decimal("0.70"), Decimal("0.50")),
        TimeOfUsePeriod("valley", "谷", time(22, 0), time(6, 0),
                        Decimal("0.30"), Decimal("0.30")),
    ]

    def __init__(self, db_client, redis_client):
        self.db = db_client
        self.redis = redis_client
        self._rate_cache: dict = {}  # station_id → TOU配置缓存
        self._load_rates()

    def _load_rates(self):
        """从数据库加载各站点的费率配置"""
        # 不同站点可能有不同的费率（如商业区 vs 郊区）
        pass

    def get_current_period(self, station_id: str, dt: datetime) -> TimeOfUsePeriod:
        """获取指定时间点的电价时段"""
        periods = self._rate_cache.get(station_id, self.DEFAULT_TOU_PERIODS)
        current_time = dt.time()

        for period in periods:
            if period.contains(current_time):
                return period

        # 默认返回平段
        return next(p for p in periods if "平" in p.name)

    def calculate_charge_cost(self, order_id: str, energy_kwh: float,
                               start_time: datetime, end_time: datetime,
                               power_kw: float, pile_type: str = "fast",
                               user_id: Optional[str] = None) -> dict:
        """
        计算充电费用

        处理跨时段充电：如果一次充电跨越峰谷时段，
        按各时段的电量分别计费

        返回：{
            total_cost, electricity_cost, service_cost,
            details: [{period, energy_kwh, electricity_price, service_price, subtotal}],
            discounts: [{type, amount}],
            demand_charge_share
        }
        """
        # 1. 切分充电时段——按电价时段分片
        time_slices = self._split_by_tou_periods(start_time, end_time)

        # 2. 按时段分配充电量（按时间比例分配）
        total_duration = (end_time - start_time).total_seconds()
        if total_duration <= 0:
            return self._empty_result()

        details = []
        total_electricity = Decimal("0")
        total_service = Decimal("0")

        for slice_start, slice_end, period in time_slices:
            slice_duration = (slice_end - slice_start).total_seconds()
            slice_ratio = slice_duration / total_duration
            slice_energy = Decimal(str(energy_kwh)) * Decimal(str(slice_ratio))

            # 快充服务费上浮20%
            service_rate = period.service_price
            if pile_type == "fast":
                service_rate = (service_rate * Decimal("1.2")).quantize(
                    Decimal("0.01"), rounding=ROUND_HALF_UP)

            slice_electricity_cost = (slice_energy * period.electricity_price).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)
            slice_service_cost = (slice_energy * service_rate).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)

            details.append({
                "period": period.name,
                "period_id": period.period_id,
                "start_time": slice_start.isoformat(),
                "end_time": slice_end.isoformat(),
                "duration_min": round(slice_duration / 60, 1),
                "energy_kwh": float(slice_energy.quantize(Decimal("0.01"))),
                "electricity_price": float(period.electricity_price),
                "service_price": float(service_rate),
                "electricity_cost": float(slice_electricity_cost),
                "service_cost": float(slice_service_cost),
                "subtotal": float(slice_electricity_cost + slice_service_cost),
            })

            total_electricity += slice_electricity_cost
            total_service += slice_service_cost

        # 3. 应用折扣
        discounts = self._apply_discounts(
            user_id, total_electricity, total_service, energy_kwh, pile_type)

        total_discount = sum(d["amount"] for d in discounts)
        total_cost = (total_electricity + total_service - total_discount).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 4. 分摊需量电费
        demand_share = self._calculate_demand_charge_share(
            power_kw, start_time, end_time)

        return {
            "order_id": order_id,
            "total_cost": float(total_cost),
            "electricity_cost": float(total_electricity),
            "service_cost": float(total_service),
            "discount_total": float(total_discount),
            "details": details,
            "discounts": discounts,
            "demand_charge_share": float(demand_share),
            "energy_kwh": energy_kwh,
            "average_price_per_kwh": float(total_cost / Decimal(str(energy_kwh)))
                if energy_kwh > 0 else 0,
        }

    def _split_by_tou_periods(self, start: datetime, end: datetime
                               ) -> List[Tuple[datetime, datetime, TimeOfUsePeriod]]:
        """
        按分时电价时段切分充电时间

        示例：19:30 - 22:30 充电
        → [19:30-21:00, 尖], [21:00-22:00, 峰], [22:00-22:30, 谷]
        """
        slices = []
        current = start

        while current < end:
            period = self.get_current_period("default", current)
            period_end = self._get_period_end(current, period)

            slice_end = min(period_end, end)
            slices.append((current, slice_end, period))
            current = slice_end

        return slices

    def _get_period_end(self, current: datetime, period: TimeOfUsePeriod) -> datetime:
        """获取当前时段的结束时间点"""
        today = current.date()
        start_dt = datetime.combine(today, period.start_time)
        end_dt = datetime.combine(today, period.end_time)

        if period.end_time <= period.start_time:
            # 跨午夜时段
            end_dt = datetime.combine(today + timedelta(days=1), period.end_time)

        if end_dt <= current:
            # 时段结束时间在当前时间之前 → 说明跨了午夜
            end_dt = datetime.combine(today + timedelta(days=1), period.end_time)

        return end_dt

    def _apply_discounts(self, user_id: Optional[str],
                          electricity_cost: Decimal, service_cost: Decimal,
                          energy_kwh: float, pile_type: str) -> List[dict]:
        """
        应用折扣规则

        折扣优先级（叠加顺序）：
        1. 会员折扣（金卡95折、银卡97折）
        2. 谷时额外优惠（谷时充电服务费9折）
        3. 优惠券抵扣
        4. 新用户首充优惠
        """
        discounts = []

        if not user_id:
            return discounts

        user = self.db.get_user(user_id)
        if not user:
            return discounts

        # 会员折扣
        member_discount_rate = {
            "platinum": Decimal("0.90"),   # 铂金9折
            "gold": Decimal("0.95"),       # 金卡95折
            "silver": Decimal("0.97"),     # 银卡97折
            "normal": Decimal("1.00"),     # 普通
        }
        rate = member_discount_rate.get(user.get("member_level", "normal"), Decimal("1.00"))
        if rate < Decimal("1.00"):
            discount_amount = ((electricity_cost + service_cost) * (Decimal("1.00") - rate)
                              ).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
            discounts.append({
                "type": "member_discount",
                "rate": float(rate),
                "amount": float(discount_amount),
                "description": f"会员{user.get('member_level')}折扣{int((1-float(rate))*100)}%",
            })

        # 优惠券
        coupons = self.db.get_available_coupons(user_id)
        for coupon in coupons:
            if coupon["type"] == "fixed" and coupon["min_amount"] <= float(electricity_cost + service_cost):
                discount_amount = Decimal(str(coupon["amount"]))
                discounts.append({
                    "type": "coupon",
                    "coupon_id": coupon["coupon_id"],
                    "amount": float(discount_amount),
                    "description": f"优惠券抵扣¥{coupon['amount']}",
                })
            elif coupon["type"] == "percentage":
                discount_amount = ((electricity_cost + service_cost) * Decimal(str(coupon["rate"]))
                                  ).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
                discounts.append({
                    "type": "coupon",
                    "coupon_id": coupon["coupon_id"],
                    "amount": float(discount_amount),
                    "description": f"优惠券{int(coupon['rate']*100)}%折扣",
                })
            # 只使用一张优惠券（最高优惠优先）
            break

        # 新用户首充优惠
        charge_count = self.db.get_user_charge_count(user_id)
        if charge_count == 0:
            first_charge_discount = Decimal("10.00")  # 首充减10元
            discounts.append({
                "type": "first_charge",
                "amount": float(first_charge_discount),
                "description": "新用户首充优惠¥10",
            })

        return discounts

    def _calculate_demand_charge_share(self, power_kw: float,
                                        start_time: datetime,
                                        end_time: datetime) -> Decimal:
        """
        计算本次充电的需量电费分摊

        需量电费按月结算：月需量电费 = max(当月最大需量, 合约需量) × 单价
        单次充电需量分摊 = (充电功率 / 当月最大需量) × 月需量电费 / 当月充电次数
        """
        # 简化计算：按功率占比分摊
        demand_config = DemandChargeConfig(
            contract_demand_kw=Decimal("2000"),    # 2MW合约需量
            demand_charge_per_kw=Decimal("38"),    # 38元/kW/月
            penalty_threshold=Decimal("2100"),
            penalty_rate=Decimal("2.0"),
        )

        # 月需量电费估算
        monthly_demand_cost = demand_config.contract_demand_kw * demand_config.demand_charge_per_kw

        # 本次充电的分摊比例
        demand_ratio = Decimal(str(power_kw)) / demand_config.contract_demand_kw

        # 按充电时长占月的比例
        charge_ratio = Decimal(str((end_time - start_time).total_seconds())) / Decimal(str(30 * 24 * 3600))

        share = (monthly_demand_cost * demand_ratio * charge_ratio).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        return share

    def _empty_result(self) -> dict:
        """空结果"""
        return {
            "total_cost": 0, "electricity_cost": 0, "service_cost": 0,
            "discount_total": 0, "details": [], "discounts": [],
            "demand_charge_share": 0, "energy_kwh": 0, "average_price_per_kwh": 0,
        }


class OccupancyFeeCalculator:
    """
    占位费计算器

    规则：
    - 充电完成后10分钟免费挪车期
    - 超过10分钟：¥1.0/分钟（快充桩）、¥0.5/分钟（慢充桩）
    - 单日占位费封顶¥200
    - 会员减免：铂金会员免费挪车期30分钟
    """

    FREE_PERIOD_MINUTES = 10
    FAST_PILE_RATE = Decimal("1.0")     # 元/分钟
    SLOW_PILE_RATE = Decimal("0.5")     # 元/分钟
    DAILY_CAP = Decimal("200.0")

    def calculate_occupancy_fee(self, pile_id: str, user_id: str,
                                 charge_complete_time: datetime,
                                 leave_time: datetime) -> dict:
        """计算占位费"""
        pile = self.db.get_pile(pile_id)
        user = self.db.get_user(user_id)

        # 免费挪车期
        free_minutes = self.FREE_PERIOD_MINUTES
        if user and user.get("member_level") == "platinum":
            free_minutes = 30  # 铂金会员30分钟免费

        # 计算占位时长
        occupancy_duration = (leave_time - charge_complete_time).total_seconds() / 60
        chargeable_minutes = max(0, occupancy_duration - free_minutes)

        if chargeable_minutes <= 0:
            return {"fee": 0, "chargeable_minutes": 0, "free_minutes": free_minutes}

        # 按桩类型选择费率
        rate = self.FAST_PILE_RATE if pile["pile_type"] == "fast" else self.SLOW_PILE_RATE

        # 计算费用
        fee = (Decimal(str(int(chargeable_minutes))) * rate).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 日封顶
        today_total = self.db.get_today_occupancy_fee(user_id)
        if today_total + fee > self.DAILY_CAP:
            fee = max(0, self.DAILY_CAP - today_total)

        return {
            "fee": float(fee),
            "chargeable_minutes": int(chargeable_minutes),
            "free_minutes": free_minutes,
            "rate": float(rate),
            "pile_type": pile["pile_type"],
            "today_total": float(today_total + fee),
            "cap_remaining": float(self.DAILY_CAP - today_total - fee),
        }


class DemandChargeManager:
    """
    需量电费管理器（月度结算）

    需量电费是充电站运营的重大成本项。
    电网公司按月收取，标准：
    - 按合约需量：38元/kW/月
    - 按实际最大需量：如果超出合约需量的105%，超出部分按2倍惩罚

    优化策略：
    - 限制站点最大功率在合约需量以内
    - 削峰填谷，降低最大需量
    - 合理设定合约需量（过高浪费、过低惩罚）
    """

    def __init__(self, db_client, redis_client):
        self.db = db_client
        self.redis = redis_client

    def record_demand(self, station_id: str, timestamp: datetime, power_kw: float):
        """记录需量数据点（每15分钟取最大值）"""
        # 计算当前15分钟时间窗口
        window_start = timestamp.replace(
            minute=(timestamp.minute // 15) * 15, second=0, microsecond=0)
        window_key = f"demand:{station_id}:{window_start.isoformat()}"

        # 记录窗口内最大功率
        current_max = self.redis.get(window_key)
        if current_max is None or power_kw > float(current_max):
            self.redis.set(window_key, str(power_kw), ex=45 * 24 * 3600)  # 保留45天

    def get_monthly_max_demand(self, station_id: str, year: int, month: int) -> float:
        """获取月度最大需量"""
        pattern = f"demand:{station_id}:{year}-{month:02d}-*"
        keys = self.redis.keys(pattern)

        max_demand = 0.0
        for key in keys:
            value = float(self.redis.get(key))
            max_demand = max(max_demand, value)

        return max_demand

    def calculate_monthly_demand_charge(self, station_id: str,
                                         year: int, month: int,
                                         contract_demand_kw: float) -> dict:
        """计算月度需量电费"""
        actual_max_demand = self.get_monthly_max_demand(station_id, year, month)
        price_per_kw = 38.0  # 元/kW/月
        penalty_threshold = contract_demand_kw * 1.05

        if actual_max_demand <= contract_demand_kw:
            # 未超出合约需量 → 按合约需量收费
            charge = contract_demand_kw * price_per_kw
            penalty = 0
        elif actual_max_demand <= penalty_threshold:
            # 超出合约需量但在105%以内 → 按实际需量收费
            charge = actual_max_demand * price_per_kw
            penalty = 0
        else:
            # 超出105% → 超出部分按2倍惩罚
            normal_charge = actual_max_demand * price_per_kw
            excess = actual_max_demand - penalty_threshold
            penalty = excess * price_per_kw  # 惩罚倍率1x（即2x-1x）
            charge = normal_charge + penalty

        return {
            "station_id": station_id,
            "year": year,
            "month": month,
            "contract_demand_kw": contract_demand_kw,
            "actual_max_demand_kw": actual_max_demand,
            "penalty_threshold_kw": penalty_threshold,
            "price_per_kw": price_per_kw,
            "base_charge": float(min(actual_max_demand, contract_demand_kw) * price_per_kw),
            "excess_charge": float(max(0, actual_max_demand - contract_demand_kw) * price_per_kw),
            "penalty": float(penalty),
            "total_charge": float(charge),
            "savings_vs_no_management": float(
                max(0, (actual_max_demand - contract_demand_kw) * price_per_kw * 2)
                if actual_max_demand > contract_demand_kw else 0
            ),
        }

    def optimize_contract_demand(self, station_id: str) -> dict:
        """
        优化合约需量设定

        基于过去6个月的需量数据，推荐最优合约需量。
        过高 → 每月多付基础费；过低 → 超限惩罚

        最优值 = 使总费用最小的合约需量
        """
        # 获取过去6个月的月度最大需量
        historical_demands = self._get_historical_demands(station_id, months=6)

        if not historical_demands:
            return {"recommended": 2000, "confidence": "low"}

        # 遍历候选合约需量，计算总费用
        min_demand = min(historical_demands) * 0.8
        max_demand = max(historical_demands) * 1.2

        best_contract = None
        best_total_cost = float('inf')

        for candidate in range(int(min_demand), int(max_demand) + 50, 50):
            total_cost = 0
            for actual_demand in historical_demands:
                if actual_demand <= candidate * 1.05:
                    cost = max(actual_demand, candidate) * 38
                else:
                    excess = actual_demand - candidate * 1.05
                    cost = actual_demand * 38 + excess * 38
                total_cost += cost

            if total_cost < best_total_cost:
                best_total_cost = total_cost
                best_contract = candidate

        current_contract = self.db.get_station(station_id)["contract_demand_kw"]
        current_cost = sum(
            self._calculate_cost(d, current_contract)
            for d in historical_demands
        )
        optimized_cost = best_total_cost

        return {
            "recommended_contract_kw": best_contract,
            "current_contract_kw": current_contract,
            "estimated_monthly_savings": float(current_cost - optimized_cost) / len(historical_demands),
            "historical_max_demand": max(historical_demands),
            "historical_avg_demand": sum(historical_demands) / len(historical_demands),
            "confidence": "high" if len(historical_demands) >= 4 else "medium",
        }

    def _get_historical_demands(self, station_id: str, months: int) -> List[float]:
        """获取历史月度最大需量"""
        demands = []
        now = datetime.utcnow()
        for i in range(1, months + 1):
            year = now.year - (1 if now.month - i <= 0 else 0)
            month = (now.month - i) % 12 or 12
            demand = self.get_monthly_max_demand(station_id, year, month)
            if demand > 0:
                demands.append(demand)
        return demands

    def _calculate_cost(self, actual_demand: float, contract: float) -> float:
        """计算单月需量电费"""
        if actual_demand <= contract * 1.05:
            return max(actual_demand, contract) * 38
        else:
            excess = actual_demand - contract * 1.05
            return actual_demand * 38 + excess * 38
```

**跨时段充电计费示例：**

```
用户 19:30 开始充电，22:30 充电结束，总充电量 45kWh

时段切分：
  19:30 - 21:00  尖段  1.5小时  占比37.5%  电量16.875kWh
  21:00 - 22:00  峰段  1.0小时  占比25.0%  电量11.25kWh
  22:00 - 22:30  谷段  0.5小时  占比12.5%  电量5.625kWh

费用计算：
  尖段：16.875 × (1.35 + 0.80×1.2) = 16.875 × 2.31 = ¥38.98
  峰段：11.25 × (1.20 + 0.70×1.2) = 11.25 × 2.04 = ¥22.95
  谷段：5.625 × (0.30 + 0.30×1.2) = 5.625 × 0.66 = ¥3.71
  合计：¥65.64

  金卡会员95折：-¥3.28
  最终：¥62.36

  平均单价：¥62.36 / 45kWh = ¥1.39/kWh
```

**需量电费成本分析：**

```
典型充电站需量电费（2MW合约）：

月度基础：2000kW × 38元/kW = ¥76,000/月

如果功率调度不到位，月度最大需量达到 2200kW（超限10%）：
  正常费用：2200 × 38 = ¥83,600
  惩罚费用：(2200 - 2100) × 38 = ¥3,800
  合计：¥87,400/月
  vs 合理调度（控制在2000kW以内）：¥76,000/月
  月度节省：¥11,400，年度节省：¥136,800

500个站 × ¥13.7万/站/年 = ¥6,850万/年的需量电费优化空间
```

## 常见陷阱（深度分析）

### 陷阱 1：不做功率调度

**后果：** 超过站点功率上限 → 变电站跳闸 → 全站断电 → 所有正在充电的车辆中断 → 安全事故。

**真实案例：** 某充电站 20 个 60kW 快充桩，总功率需求 1.2MW，站点变压器容量 2MW。夏季高温时站点空调耗电 300kW → 可用功率只剩 1.7MW → 15 桩同时快充就超过限额 → 未做调度 → 变压器跳闸 → 15 辆车充电中断 → 用户投诉 + 安全隐患。

**解决方案：** 功率调度器实时监控总功率 + 站点辅助设备耗电余量 + 降功率或排队。

### 陷阱 2：预约无宽限期

**后果：** 用户迟到 5 分钟就取消 → 预约体验差 → 用户不愿预约 → 桩利用率低。

**解决方案：** 15 分钟宽限期 + 宽限期到期自动释放。但宽限期不能太长——30 分钟宽限期意味着桩被锁定 30 分钟，高峰期浪费严重。

### 陷阱 3：不引导谷时充电

**后果：** 峰时充电桩拥挤（排队 30 分钟）、谷时空闲（利用率 < 20%） → 整体利用率低 → 运营亏损。

**成本分析：** 1 万桩 × 平均 60kW × 峰时 4 小时 × ¥1.2/度 = ¥288 万/天电费。如果谷时利用率从 20% 提升到 60%，每天节省约 ¥50 万。

**解决方案：** 电价引导 + 谷时折扣激励 + 占位费疏导峰时拥挤。

### 陷阱 4：充完不挪车无罚则

**后果：** 用户充完后占着桩位 1-2 小时 → 后续用户无法使用 → 约纷投诉。实测数据：无占位费时平均占位时间 47 分钟，有占位费时降至 8 分钟。

**解决方案：** 10 分钟免费挪车期 + 超时收取占位费（¥0.5/分钟）。占位费通过小程序推送+短信通知，用户不缴费则下次无法预约。

### 陷阱 5：功率调整指令丢失

**场景：** 云端下发降功率指令 → MQTT 消息丢失 → 桩继续满功率充电 → 站点超功率 → 跳闸。

**解决方案：** 关键指令使用 QoS 2 + 指令 ACK 确认 + 超时重试。如果 3 次下发失败 → 标记桩故障 → 通知用户换桩。

### 陷阱 6：降功率充电不通知用户

**后果：** 用户期望 40 分钟快充完成，实际 80 分钟 → 投诉"充电慢"。用户不知道功率被降低，以为桩坏了。

**解决方案：** 降功率时立即通知用户（App推送 + 桩屏幕显示），说明原因和预计时间，给出选择：继续降功率等待或排队等满功率。

## 异常场景完整演练

**场景 1：站点变压器跳闸**

```
触发：总功率超过2MW（15个桩快充 + 空调 + 照明）
处理：
  1. 变电站跳闸 → 全站断电 → 所有桩离线
  2. MQTT心跳检测 → 标记全站offline
  3. 通知所有正在充电的用户："充电站故障，请移至附近站点"
  4. 自动取消全站预约
  5. 电力恢复后 → 桩逐个上线 → 功率调度器限制初始同时充电数（≤10桩）
  6. 逐步放开充电桩数量，避免再次跳闸

关键：恢复后不能让所有桩同时启动充电（冲击电流），
     必须逐个启动，间隔至少5秒
```

**场景 2：单桩故障（正在充电中）**

```
触发：桩上报fault（过温/通信异常/硬件故障）
处理：
  1. 功率调度器释放该桩的功率预算
  2. 查找站内可用备用桩
  3. 自动切换：更新订单的pile_id → 向新桩下发充电指令
  4. 通知用户："充电桩{X}故障，已自动切换到{Y}桩"
  5. 如果站内无备用桩 → 退还本次充电费用 + 赠送优惠券
```

**场景 3：大规模预约取消（暴雨天气）**

```
触发：暴雨 → 大量用户取消预约 → 空闲桩突然增多
处理：
  1. 预约取消 → 释放桩位和功率预算
  2. 通知排队中的用户："空出快充桩，是否立即充电？"
  3. 动态降低充电费用（临时折扣）吸引附近用户
  4. 暴雨过后 → 预约恢复 → 逐步恢复正常定价
```

## 性能与成本分析

**功率调度性能：**

| 操作 | 延迟 | QPS |
|------|------|-----|
| 请求充电（查缓存） | < 5ms | 500/s |
| 功率升级（MQTT下发） | < 50ms | 100/s |
| 排队唤醒 | < 100ms | 50/s |
| 心跳处理 | < 2ms | 1000/s |

**Redis 资源需求：**
- 站点功率状态：1 万桩 × 约 100 字节/桩 = 1MB
- 预约信息：约 5000 活跃预约 × 约 200 字节 = 1MB
- 排队队列：约 500 排队 × 约 100 字节 = 50KB
- 总计约 2MB，单节点 Redis 即可

**MQTT 连接管理：**
- 1 万桩 × 每桩 1 个 MQTT 连接 = 1 万连接
- EMQX 单节点支持 10 万连接 → 1 节点即可
- 消息吞吐：1000 次/秒上报 + 500 次/秒指令 = 1500 条/秒

**收入估算：**
- 日均 5 万次充电 × 平均 ¥40/次 = ¥200 万/天
- 谷时充电引导提升利用率 → 日增收约 ¥15 万
- 占位费收入 → 日约 ¥2 万
- 年营收约 ¥7.9 亿

## 延伸思考

- **V2G（车到网）**：电动车在谷时充电、峰时放电 → 既赚钱又削峰填谷。需要双向充电桩和电池管理协议。收益估算：每辆车峰时放电 30kWh × (¥1.2 - ¥0.3) = ¥27/天，月收益约 ¥800。
- **自动驾驶代客充电**：车辆自动驶向空闲桩，充完自动驶离 → 利用率大幅提升 + 无占位问题。核心挑战：场内自动驾驶的精度（±10cm 对准充电口）。
- **光伏+储能**：充电站屋顶安装光伏 + 储能电池 → 白天光伏充电，夜间储能放电 → 减少电网依赖。500㎡屋顶光伏 → 日发电约 300kWh → 可供 5 车快充。
- **功率预测与预调度**：根据预约数据预测未来 1 小时的功率需求 → 提前调度（如通知降功率桩用户稍后充电）→ 避免实时调度带来的用户体验降级。
## 动态电价策略完整实现

```python
class DynamicPricingService:
    """动态电价：根据电网负荷和时段调整"""

    PRICE_SCHEDULE = {
        "peak": {"hours": [8,9,10,11,17,18,19,20], "rate": 1.5, "name": "高峰时段"},
        "normal": {"hours": [7,12,13,14,15,16,21,22], "rate": 1.0, "name": "平段"},
        "off_peak": {"hours": [0,1,2,3,4,5,6,23], "rate": 0.5, "name": "低谷时段"},
    }

    def calculate_price(self, station_id, start_time, estimated_duration_minutes):
        """计算充电费用（考虑时段变化）"""
        total_cost = 0
        power_kw = 60  # 60kW 快充
        current_time = start_time
        remaining_minutes = estimated_duration_minutes

        while remaining_minutes > 0:
            hour = current_time.hour
            # 确定当前时段
            rate_type = self._get_rate_type(hour, station_id)
            rate = self.PRICE_SCHEDULE[rate_type]["rate"]

            # 计算本时段充电费用
            # 可能跨越时段 → 拆分
            next_change_hour = self._next_rate_change_hour(hour)
            minutes_in_current_rate = min(
                remaining_minutes,
                (next_change_hour - hour) * 60 if next_change_hour > hour
                else remaining_minutes
            )

            kwh_in_segment = power_kw * minutes_in_current_rate / 60
            cost_in_segment = kwh_in_segment * rate
            total_cost += cost_in_segment

            remaining_minutes -= minutes_in_current_rate
            current_time += timedelta(minutes=minutes_in_current_rate)

        return {
            "total_cost": round(total_cost, 2),
            "total_kwh": round(power_kw * estimated_duration_minutes / 60, 2),
            "breakdown": self._get_cost_breakdown(start_time, estimated_duration_minutes)
        }

    def _get_rate_type(self, hour, station_id):
        """获取当前时段（考虑电网实时负荷）"""
        base_type = "off_peak" if hour in self.PRICE_SCHEDULE["off_peak"]["hours"] \
            else "peak" if hour in self.PRICE_SCHEDULE["peak"]["hours"] else "normal"

        # 电网负荷修正：负荷 > 90% → 价格上浮 20%
        grid_load = self.grid_client.get_load(station_id)
        if grid_load > 0.9:
            return "peak"  # 即使低谷时段，电网紧张也按高峰计价

        return base_type
```

## 充电桩预约排队系统

```python
class ChargingReservationService:
    """充电桩预约排队"""

    def reserve(self, user_id, station_id, preferred_time):
        """预约充电"""
        # 1. 检查充电桩可用性
        available_ports = self.db.query(
            "SELECT * FROM charging_ports WHERE station_id = %s "
            "AND status = 'available' "
            "AND next_available_time <= %s",
            station_id, preferred_time)
        if not available_ports:
            # 无可用桩 → 加入排队
            queue_position = self.redis.incr(f"queue:{station_id}")
            self.redis.set(f"queue_user:{station_id}:{queue_position}", user_id)
            return {"status": "queued", "position": queue_position}

        # 2. 分配充电桩
        port = available_ports[0]
        self.db.update("charging_ports",
            {"status": "reserved", "reserved_by": user_id,
             "reserved_at": preferred_time},
            {"id": port["id"]})

        return {"status": "reserved", "port_id": port["id"],
                "estimated_cost": self.pricing.calculate_price(
                    station_id, preferred_time, 45)}

    def process_queue(self, station_id):
        """处理排队：充电桩空闲时通知下一个用户"""
        next_user = self.redis.get(f"queue_user:{station_id}:1")
        if next_user:
            # 通知用户
            self.notify(next_user, f"充电桩已空闲，请在 10 分钟内到达")
            # 设置 10 分钟超时
            self.redis.setex(f"queue_timeout:{station_id}:1", 600, next_user)
```

## 异常场景补充

### 场景：电网负荷告警

```
触发：区域电网负荷 > 95% → 需要限制充电功率
检测：
  1. 电网负荷实时监控 > 85% → 告警
  2. > 95% → 严重告警 + 限制充电
处理：
  1. 负荷 85-95% → 降低快充功率（60kW → 30kW）
  2. 负荷 > 95% → 只允许慢充（7kW）+ 限制同时充电数量
  3. 负荷 > 98% → 暂停所有非紧急充电
  4. 恢复后逐步放开
预防：电网联动 + 动态功率限制 + 用户提前通知
```

### 场景：充电桩通信中断

```
触发：充电桩 MQTT 连接断开 → 无法获取实时状态
检测：
  1. 心跳超时 > 30 秒 → 告警
  2. 充电桩状态变为 unknown
处理：
  1. 标记充电桩为 "disconnected"
  2. 如果正在充电 → 使用本地计量（桩端独立计费）
  3. 恢复后 → 同步本地计量数据到云端
预防：桩端独立计费能力 + MQTT 集群 + 重连机制
```

## 充电计费引擎完整实现

```python
class ChargingBillingEngine:
    """充电计费引擎：按时段+电量+服务费"""

    def calculate_bill(self, session_id):
        """计算充电费用"""
        session = self.db.get_charging_session(session_id)

        # 1. 获取充电电量
        kwh_consumed = session["end_kwh"] - session["start_kwh"]

        # 2. 按时段拆分计费
        segments = self._split_by_price_period(
            session["start_time"], session["end_time"])

        total_cost = 0
        bill_items = []
        for seg in segments:
            period_kwh = kwh_consumed * (seg["duration_minutes"] /
                session["duration_minutes"])
            electricity_cost = period_kwh * seg["rate"]
            service_cost = period_kwh * seg["service_rate"]

            bill_items.append({
                "period": seg["name"],
                "hours": seg["duration_minutes"] / 60,
                "kwh": round(period_kwh, 2),
                "electricity_rate": seg["rate"],
                "electricity_cost": round(electricity_cost, 2),
                "service_rate": seg["service_rate"],
                "service_cost": round(service_cost, 2),
                "subtotal": round(electricity_cost + service_cost, 2)
            })
            total_cost += electricity_cost + service_cost

        # 3. 会员折扣
        user = self.db.get_user(session["user_id"])
        if user["membership"] == "premium":
            discount = 0.9  # 9 折
            total_cost *= discount

        return {
            "session_id": session_id,
            "total_kwh": round(kwh_consumed, 2),
            "total_cost": round(total_cost, 2),
            "items": bill_items,
            "discount": "10%" if user["membership"] == "premium" else None
        }

    def _split_by_price_period(self, start, end):
        """按时段拆分"""
        periods = []
        current = start
        while current < end:
            hour = current.hour
            if hour in range(0, 7):
                rate, name = 0.3, "低谷"
            elif hour in range(8, 11) or hour in range(17, 21):
                rate, name = 1.2, "高峰"
            else:
                rate, name = 0.8, "平段"

            next_hour = (current.replace(minute=0, second=0) +
                        timedelta(hours=1))
            segment_end = min(next_hour, end)
            duration = (segment_end - current).total_seconds() / 60

            periods.append({
                "name": name, "rate": rate,
                "service_rate": 0.5,  # 服务费 0.5 元/kWh
                "duration_minutes": duration
            })
            current = segment_end
        return periods
```

## 充电站运维管理

```python
class StationMaintenanceService:
    """充电站运维管理"""

    def schedule_maintenance(self, station_id, maintenance_type):
        """安排维护"""
        # 1. 查找空闲时段（充电量最少的时段）
        idle_period = self._find_idle_period(station_id)
        if not idle_period:
            raise NoMaintenanceWindowError("充电站无空闲时段")

        # 2. 逐个端口维护（不关闭整个站点）
        ports = self.db.query(
            "SELECT * FROM charging_ports WHERE station_id = %s AND status = 'available'",
            station_id)

        for port in ports:
            self.db.insert("maintenance_schedule", {
                "port_id": port["id"],
                "station_id": station_id,
                "type": maintenance_type,
                "scheduled_start": idle_period["start"],
                "scheduled_end": idle_period["end"],
                "status": "scheduled"
            })

    def detect_anomaly(self, port_id):
        """检测充电桩异常"""
        metrics = self._get_port_metrics(port_id)
        anomalies = []

        # 充电效率下降
        if metrics["efficiency"] < 0.85:
            anomalies.append({
                "type": "low_efficiency",
                "value": metrics["efficiency"],
                "threshold": 0.85,
                "action": "inspect_cable_and_connector"
            })

        # 故障率升高
        if metrics["error_rate_7d"] > 0.05:
            anomalies.append({
                "type": "high_error_rate",
                "value": metrics["error_rate_7d"],
                "threshold": 0.05,
                "action": "schedule_maintenance"
            })

        return anomalies
```

## 异常场景补充

### 场景：充电中断无法重启

```
触发：充电到 50% 中断 → 用户尝试重启 → 充电桩报错
检测：
  1. 充电会话状态变为 interrupted
  2. 用户 3 分钟内重新扫码 → 重启请求
处理：
  1. 检查中断原因：过温/过流/通信故障
  2. 过温 → 等待 10 分钟冷却后自动恢复
  3. 通信故障 → 切换到备用通信通道
  4. 硬件故障 → 标记端口故障，引导用户换桩
  5. 已充电量正常计费
预防：充电中断自动诊断 + 用户引导换桩
```

### 场景：多车同时充电导致电网过载

```
触发：10 辆车同时快充 → 局部电网负荷 > 变压器容量
检测：
  1. 变压器负荷 > 90% → 告警
  2. 电压下降 > 5% → 严重告警
处理：
  1. 限制后续快充功率（60kW → 30kW）
  2. 新请求排队等待
  3. 优先保障已在充电的车辆
  4. 负荷恢复后逐步放开
预防：电网容量联动 + 动态功率分配 + 充电限流
```

## 充电桩互联互通平台完整实现

```python
class InteropPlatformService:
    """充电桩互联互通：多运营商接入 + 统一协议"""

    def register_operator(self, operator_id, api_config):
        """注册充电运营商"""
        self.db.insert("charging_operators", {
            "operator_id": operator_id,
            "name": api_config["name"],
            "api_endpoint": api_config["endpoint"],
            "auth_type": api_config["auth_type"],
            "protocol_version": api_config.get("protocol_version", "2.0"),
            "status": "active",
            "registered_at": now()
        })

        # 同步该运营商的充电站数据
        stations = self.operator_client.get_stations(operator_id)
        for station in stations:
            self._sync_station(operator_id, station)

    def unified_query(self, user_id, lat, lng, radius_km=5):
        """统一查询附近充电站（跨运营商）"""
        # 从各运营商查询
        operators = self.db.query(
            "SELECT * FROM charging_operators WHERE status = 'active'")

        all_stations = []
        for op in operators:
            try:
                stations = self.operator_client.query_nearby(
                    op["operator_id"], lat, lng, radius_km)
                for s in stations:
                    s["operator_id"] = op["operator_id"]
                    s["operator_name"] = op["name"]
                all_stations.extend(stations)
            except Exception as e:
                self.log.warning(f"运营商 {op['operator_id']} 查询失败: {e}")

        # 统一排序：距离优先
        for s in all_stations:
            s["distance_km"] = self.haversine(lat, lng, s["lat"], s["lng"])
        all_stations.sort(key=lambda x: x["distance_km"])

        return all_stations

    def unified_start_charging(self, user_id, station_id, port_id):
        """统一启动充电（跨运营商）"""
        station = self.db.get_station(station_id)
        operator_id = station["operator_id"]

        # 调用运营商 API 启动充电
        result = self.operator_client.start_charging(
            operator_id, {
                "port_id": port_id,
                "user_id": user_id,
                "connector_type": self._get_user_connector(user_id)
            })

        # 创建统一的充电会话
        session_id = str(uuid4())
        self.db.insert("charging_sessions", {
            "session_id": session_id,
            "user_id": user_id,
            "operator_id": operator_id,
            "station_id": station_id,
            "port_id": port_id,
            "operator_session_id": result["session_id"],
            "status": "charging",
            "start_time": now(),
            "start_kwh": result.get("start_kwh", 0)
        })

        return {"session_id": session_id, "operator": operator_id}
```

## 充电站收益分析

```python
class StationProfitAnalyzer:
    """充电站收益分析"""

    def analyze(self, station_id, period="monthly"):
        """分析充电站收益"""
        sessions = self.db.query(
            "SELECT * FROM charging_sessions WHERE station_id = %s "
            "AND status = 'completed' AND end_time >= %s",
            station_id,
            now() - timedelta(days=30 if period == "monthly" else 7))

        total_kwh = sum(s["kwh_consumed"] for s in sessions)
        total_revenue = sum(s["total_cost"] for s in sessions)
        total_electricity_cost = sum(
            s["kwh_consumed"] * self._get_electricity_rate(s) for s in sessions)

        # 利用率
        ports = self.db.count("charging_ports", station_id=station_id)
        total_hours = (now() - (now() - timedelta(days=30))).total_seconds() / 3600
        used_hours = sum((s["end_time"] - s["start_time"]).total_seconds() / 3600
            for s in sessions)
        utilization_rate = used_hours / (total_hours * ports)

        return {
            "station_id": station_id,
            "period": period,
            "total_sessions": len(sessions),
            "total_kwh": round(total_kwh, 1),
            "total_revenue": round(total_revenue, 2),
            "electricity_cost": round(total_electricity_cost, 2),
            "gross_profit": round(total_revenue - total_electricity_cost, 2),
            "utilization_rate": round(utilization_rate, 3),
            "avg_session_kwh": round(total_kwh / len(sessions), 1) if sessions else 0,
        }
```

## 异常场景补充

### 场景：运营商 API 不可用

```
触发：某运营商 API 宕机 → 该运营商充电站无法查询/启动
检测：
  1. API 调用超时 > 5 秒 → 标记不可用
  2. 连续失败 > 3 次 → 从查询结果中排除
处理：
  1. 查询时排除不可用运营商 → 返回其他运营商结果
  2. 用户已在该运营商充电 → 本地记录，稍后同步
  3. 运营商恢复后 → 同步积压的充电记录
预防：多运营商冗余 + API 健康检查 + 本地缓存
```

### 场景：充电费用争议

```
触发：用户认为充电费用不正确 → 发起争议
检测：用户提交费用争议 → 客服工单
处理：
  1. 调取充电会话详细数据（电量、时段、费率）
  2. 重新计算费用（使用原始数据）
  3. 计算结果与收费一致 → 解释明细
  4. 计算结果不一致 → 退还差额
预防：费用明细透明化 + 实时计费预览 + 争议快速处理
```

## 充电桩远程诊断完整实现

```python
class RemoteDiagnosticsService:
    """充电桩远程诊断与修复"""

    def diagnose(self, port_id):
        """远程诊断充电桩"""
        port = self.db.get_port(port_id)
        checks = []

        # 1. 通信检查
        comm_ok = self.mqtt_client.ping(port["device_id"])
        checks.append({"name": "通信", "status": "ok" if comm_ok else "fail"})

        # 2. 电表读数
        meter = self.mqtt_client.get_meter(port["device_id"])
        if meter:
            checks.append({"name": "电表", "status": "ok", "voltage": meter["voltage"]})
            if meter["voltage"] < 200 or meter["voltage"] > 250:
                checks[-1]["status"] = "abnormal"
                checks[-1]["detail"] = f"电压异常: {meter['voltage']}V"
        else:
            checks.append({"name": "电表", "status": "no_response"})

        # 3. 温度检查
        temp = self.mqtt_client.get_temperature(port["device_id"])
        if temp and temp > 85:
            checks.append({"name": "温度", "status": "overheat",
                           "value": f"{temp}°C"})
        elif temp:
            checks.append({"name": "温度", "status": "ok", "value": f"{temp}°C"})

        # 4. 连接器检查
        connector = self.mqtt_client.get_connector_status(port["device_id"])
        checks.append({"name": "连接器", "status": connector.get("status", "unknown")})

        # 综合诊断
        all_ok = all(c["status"] == "ok" for c in checks)
        return {
            "port_id": port_id,
            "overall_status": "healthy" if all_ok else "faulty",
            "checks": checks,
            "recommended_action": self._recommend_action(checks)
        }

    def _recommend_action(self, checks):
        """推荐修复动作"""
        for check in checks:
            if check["name"] == "通信" and check["status"] == "fail":
                return "重启通信模块 → 仍失败则派现场工程师"
            if check["name"] == "温度" and check["status"] == "overheat":
                return "远程降功率运行 → 30分钟后复查温度"
            if check["name"] == "连接器" and check["status"] == "locked":
                return "发送解锁指令 → 仍锁定则现场更换连接器"
        return "无需操作"
```

## 充电用户会员体系

```python
class ChargingMembershipService:
    """充电会员体系"""

    TIERS = {
        "free": {"monthly_fee": 0, "discount": 0, "free_kwh": 0},
        "silver": {"monthly_fee": 29.9, "discount": 0.05, "free_kwh": 20},
        "gold": {"monthly_fee": 59.9, "discount": 0.10, "free_kwh": 50},
        "platinum": {"monthly_fee": 99.9, "discount": 0.15, "free_kwh": 100},
    }

    def apply_membership_discount(self, user_id, kwh, base_cost):
        """应用会员折扣"""
        user = self.db.get_user(user_id)
        tier = user.get("membership_tier", "free")
        benefits = self.TIERS[tier]

        # 免费电量
        free_kwh_used = self.redis.get(f"free_kwh_used:{user_id}:{now().month}")
        remaining_free = max(0, benefits["free_kwh"] - int(free_kwh_used or 0))
        actual_free = min(remaining_free, kwh)

        # 计算费用
        paid_kwh = kwh - actual_free
        per_kwh_cost = base_cost / kwh
        paid_cost = paid_kwh * per_kwh_cost * (1 - benefits["discount"])
        free_cost = actual_free * per_kwh_cost

        # 更新已用免费电量
        self.redis.incrbyfloat(f"free_kwh_used:{user_id}:{now().month}", actual_free)

        return {
            "tier": tier,
            "total_kwh": kwh,
            "free_kwh": actual_free,
            "paid_kwh": paid_kwh,
            "discount_rate": benefits["discount"],
            "savings": round(base_cost - paid_cost, 2),
            "final_cost": round(paid_cost, 2)
        }
```

## 异常场景补充

### 场景：远程诊断指令执行失败

```
触发：远程诊断指令下发后无响应 → 无法判断故障原因
检测：
  1. 指令下发后 10 秒无响应 → 超时
  2. 重试 3 次仍失败 → 远程诊断不可用
处理：
  1. 尝试备用通信通道（MQTT → HTTP）
  2. 仍失败 → 标记为"需现场诊断"
  3. 创建现场维修工单
预防：多通信通道 + 远程诊断超时保护 + 自动创建工单
```

### 场景：会员权益计算错误

```
触发：免费电量扣减异常 → 用户被多收费
检测：
  1. 用户投诉费用不正确 → 检查会员权益
  2. 免费电量计数 > 月度额度 → 计数溢出
处理：
  1. 重新计算当月所有充电费用
  2. 差额退还
  3. 修复计数逻辑（使用 INCRBYFLOAT 原子操作）
预防：原子计数 + 每日校验 + 费用争议快速处理
```

## 充电站选址算法完整实现

```python
class StationSiteSelector:
    """充电站选址：加权评分 + ROI 计算"""

    WEIGHTS = {
        "traffic_density": 0.30,   # 交通密度
        "grid_capacity": 0.20,     # 电网容量
        "competitor_proximity": 0.15,  # 竞品距离
        "land_cost": 0.15,         # 土地成本
        "zoning_compliance": 0.10, # 规划合规
        "visibility": 0.10,        # 可见性
    }

    def evaluate_site(self, location):
        """评估选址评分"""
        scores = {}

        # 1. 交通密度：日均车流量
        traffic = self.traffic_api.get_daily_volume(location)
        scores["traffic_density"] = self._normalize(traffic, 5000, 50000)

        # 2. 电网容量：变压器剩余容量
        grid = self.grid_api.get_available_capacity(location)
        scores["grid_capacity"] = self._normalize(grid, 0, 500)  # kW

        # 3. 竞品距离：最近充电站距离（越远越好）
        nearest = self._find_nearest_station(location)
        competitor_score = self._normalize(nearest, 0, 10) if nearest else 100
        scores["competitor_proximity"] = competitor_score

        # 4. 土地成本
        land_cost = self.real_estate_api.get_price(location)
        scores["land_cost"] = 100 - self._normalize(land_cost, 50, 500)  # 便宜=好

        # 5. 规划合规
        zoning = self.zoning_api.check_compliance(location, "commercial")
        scores["zoning_compliance"] = 100 if zoning["allowed"] else 0

        # 6. 可见性：主干道距离
        main_road_dist = self.map_api.get_nearest_main_road_distance(location)
        scores["visibility"] = 100 - self._normalize(main_road_dist, 0, 2000)

        # 加权总分
        total = sum(scores[k] * self.WEIGHTS[k] for k in scores)

        return {"location": location, "scores": scores,
                "total_score": round(total, 1)}

    def calculate_roi(self, location, config):
        """ROI 计算：5 年投资回报"""
        # 投资成本
        build_cost = config["port_count"] * 50000 + config["land_cost"]  # 每端口 5 万
        annual_operating = 80000  # 年运营成本

        # 收入预测
        daily_sessions = config["port_count"] * 8  # 每端口日均 8 次充电
        avg_kwh_per_session = 30
        avg_revenue_per_kwh = 1.2  # 电费+服务费
        annual_revenue = daily_sessions * avg_kwh_per_session * avg_revenue_per_kwh * 365

        # 5 年 ROI
        total_cost = build_cost + annual_operating * 5
        total_revenue = annual_revenue * 5
        roi = (total_revenue - total_cost) / total_cost * 100

        return {
            "build_cost": build_cost,
            "annual_operating": annual_operating,
            "annual_revenue": round(annual_revenue, 0),
            "5_year_roi": round(roi, 1),
            "payback_years": round(build_cost / (annual_revenue - annual_operating), 1)
        }

    def _normalize(self, value, min_val, max_val):
        """归一化到 0-100"""
        return max(0, min(100, (value - min_val) / (max_val - min_val) * 100))

    def generate_expansion_plan(self, budget, target_coverage_km=5):
        """生成扩张计划（预算约束）"""
        # 1. 找到覆盖缺口
        gaps = self._detect_coverage_gaps(target_coverage_km)

        # 2. 评估每个缺口位置
        candidates = []
        for gap in gaps:
            score = self.evaluate_site(gap["center"])
            roi = self.calculate_roi(gap["center"], {
                "port_count": 8, "land_cost": score["scores"]["land_cost"] * 3
            })
            candidates.append({
                "location": gap["center"],
                "score": score["total_score"],
                "roi": roi["5_year_roi"],
                "build_cost": roi["build_cost"],
                "gap_demand": gap["estimated_demand"]
            })

        # 3. 背包问题：预算内最大化覆盖
        candidates.sort(key=lambda c: c["roi"], reverse=True)
        selected = []
        remaining_budget = budget
        for c in candidates:
            if c["build_cost"] <= remaining_budget:
                selected.append(c)
                remaining_budget -= c["build_cost"]

        return {
            "total_candidates": len(candidates),
            "selected": len(selected),
            "total_budget_used": budget - remaining_budget,
            "stations": selected
        }

    def _detect_coverage_gaps(self, target_km):
        """检测覆盖缺口（现有充电站 5km 外的区域）"""
        # 获取城市内所有现有充电站
        stations = self.db.query("SELECT lat, lng FROM charging_stations")

        # 将城市划分为网格，找出距离最近的充电站 > 5km 的网格
        city_bounds = self.map_api.get_city_bounds()
        grid_size = 0.5  # 500m 网格
        gaps = []

        for lat in range(int(city_bounds["min_lat"] * 2), int(city_bounds["max_lat"] * 2)):
            for lng in range(int(city_bounds["min_lng"] * 2), int(city_bounds["max_lng"] * 2)):
                center = {"lat": lat / 2, "lng": lng / 2}
                nearest_dist = min(
                    self.haversine(center["lat"], center["lng"], s["lat"], s["lng"])
                    for s in stations) if stations else float('inf')
                if nearest_dist > target_km:
                    # 估算需求（人口密度 × 电动车渗透率）
                    demand = self.census_api.get_ev_count(center) * 0.3
                    if demand > 50:  # 50 辆以上电动车才有建站价值
                        gaps.append({"center": center, "estimated_demand": demand,
                                     "nearest_station_km": nearest_dist})

        return sorted(gaps, key=lambda g: g["estimated_demand"], reverse=True)
```

## 用户行为分析

```python
class ChargingBehaviorAnalyzer:
    """充电行为分析：模式聚类 + 流失预测 + 个性化推荐"""

    def cluster_users(self):
        """用户充电模式聚类（K-means）"""
        features = self.db.query(
            "SELECT user_id, AVG(kwh_consumed) as avg_kwh, "
            "AVG(HOUR(start_time)) as avg_hour, "
            "AVG(duration_minutes) as avg_duration, "
            "COUNT(DISTINCT station_id) as station_diversity, "
            "COUNT(*) as monthly_sessions "
            "FROM charging_sessions "
            "WHERE start_time > NOW() - INTERVAL 90 DAY "
            "GROUP BY user_id")

        # K-means 聚类
        from sklearn.cluster import KMeans
        X = [[f["avg_kwh"], f["avg_hour"], f["avg_duration"],
              f["station_diversity"], f["monthly_sessions"]] for f in features]
        kmeans = KMeans(n_clusters=4, random_state=42)
        labels = kmeans.fit_predict(X)

        CLUSTER_NAMES = {
            0: "通勤型",   # 固定时段、中等电量、低多样性
            1: "周末型",   # 非固定时段、大电量、中等多样性
            2: "长途型",   # 大电量、高时长、高多样性
            3: "偶尔型",   # 低频次、随机时段
        }

        for i, f in enumerate(features):
            self.db.update("users",
                {"charging_pattern": CLUSTER_NAMES[labels[i]]},
                {"id": f["user_id"]})

        return {name: sum(1 for l in labels if l == k)
                for k, name in CLUSTER_NAMES.items()}

    def predict_churn(self, user_id):
        """流失预测（7 天未充电 → 流失风险）"""
        last_charge = self.db.query_one(
            "SELECT MAX(end_time) as last FROM charging_sessions "
            "WHERE user_id = %s", user_id)["last"]

        if not last_charge:
            return {"risk": "high", "reason": "never_charged"}

        days_since = (now() - last_charge).days
        if days_since > 30:
            return {"risk": "very_high", "reason": f"{days_since}天未充电"}
        elif days_since > 14:
            return {"risk": "high", "reason": f"{days_since}天未充电"}
        elif days_since > 7:
            return {"risk": "medium", "reason": f"{days_since}天未充电"}
        return {"risk": "low"}
```

## 异常场景补充

### 场景：选址数据不足导致 ROI 偏低

```
触发：新建充电站实际车流量远低于预测 → ROI 不达预期
检测：
  1. 月营收 < 预测 60% → ROI 不达标
  2. 日均充电次数 < 预测 50% → 车流量不足
处理：
  1. 分析原因：数据预测误差 / 竞品分流 / 规划变更
  2. 优化：增加可见性（路牌引导）+ 会员优惠引流
  3. 如果持续不达标 → 考虑搬迁或减少端口数量
预防：选址数据多源验证 + 竞品影响评估 + 建站后持续监控
```

### 场景：扩张计划超出预算

```
触发：覆盖缺口太多 → 优质选址总成本 > 年度预算
检测：扩张计划总成本 > 预算
处理：
  1. 按 ROI 排序 → 优先选择高回报站点
  2. 减少单站端口数量（降低建设成本）
  3. 分期建设：第一期覆盖最紧迫缺口，后续逐年扩张
  4. 引入第三方投资（合作建站）
预防：分期扩张 + ROI 排序 + 第三方合作
```

## 充电站选址规划完整实现（深度版）

```python
class StationSitePlanningService:
    """充电站选址：选址算法 → ROI 计算 → 合规检查 → 并网申请 → 站点设计"""

    # ---- 1. 选址算法 ----

    SITE_WEIGHTS = {
        "traffic_density": 0.30,   # 交通密度
        "grid_capacity": 0.20,     # 电网容量
        "competitor_proximity": 0.20,  # 竞品距离
        "land_cost": 0.15,         # 地价
        "ev_penetration": 0.15,    # 区域电动车渗透率
    }

    def find_optimal_sites(self, city_id, target_count=5):
        """为城市推荐最优选址"""
        # 获取候选地块
        candidate_parcels = self.db.query(
            "SELECT * FROM land_parcels WHERE city_id = %s "
            "AND zoning_type IN ('commercial', 'industrial', 'mixed_use') "
            "AND area_sqm >= 500", city_id)

        scored_sites = []
        for parcel in candidate_parcels:
            scores = self._evaluate_site(parcel)
            if scores["total_score"] >= 60:  # 最低门槛
                scored_sites.append({
                    "parcel_id": parcel["parcel_id"],
                    "address": parcel["address"],
                    "lat": parcel["lat"],
                    "lng": parcel["lng"],
                    "area_sqm": parcel["area_sqm"],
                    **scores
                })

        # 按总分排序
        scored_sites.sort(key=lambda x: x["total_score"], reverse=True)

        # 去重：推荐站点之间至少相距 3km
        selected = []
        for site in scored_sites:
            too_close = any(
                self.haversine(site["lat"], site["lng"], s["lat"], s["lng"]) < 3
                for s in selected)
            if not too_close:
                selected.append(site)
            if len(selected) >= target_count:
                break

        return selected

    def _evaluate_site(self, parcel):
        """综合评估选址"""
        scores = {}

        # 交通密度评分（周边 2km 范围日均车流量）
        traffic_flow = self.traffic_api.get_daily_flow(
            parcel["lat"], parcel["lng"], radius_km=2)
        traffic_score = min(100, traffic_flow / 500)  # 5 万车次/天 = 100 分
        scores["traffic_density"] = round(traffic_score, 1)

        # 电网容量评分（变电站距离和可用容量）
        grid_info = self.grid_api.query_nearest_substation(
            parcel["lat"], parcel["lng"])
        if grid_info["distance_km"] <= 1 and grid_info["available_mw"] >= 2:
            grid_score = 100
        elif grid_info["distance_km"] <= 3 and grid_info["available_mw"] >= 1:
            grid_score = 70
        elif grid_info["distance_km"] <= 5:
            grid_score = 40
        else:
            grid_score = 10
        scores["grid_capacity"] = grid_score

        # 竞品距离评分（3km 内竞品站越少越好）
        nearby_stations = self.db.query(
            "SELECT * FROM charging_stations WHERE status = 'active' "
            "AND ST_Distance_Sphere(location, ST_Point(%s, %s)) <= 3000",
            parcel["lng"], parcel["lat"])
        competitor_count = len(nearby_stations)
        if competitor_count == 0:
            comp_score = 100
        elif competitor_count == 1:
            comp_score = 70
        elif competitor_count <= 3:
            comp_score = 40
        else:
            comp_score = 10
        scores["competitor_proximity"] = comp_score

        # 地价评分（越便宜越好）
        land_price = parcel.get("price_per_sqm", 10000)
        if land_price <= 3000:
            land_score = 100
        elif land_price <= 5000:
            land_score = 80
        elif land_price <= 8000:
            land_score = 60
        elif land_price <= 12000:
            land_score = 40
        else:
            land_score = 20
        scores["land_cost"] = land_score

        # 电动车渗透率评分
        ev_rate = self.db.query_one(
            "SELECT ev_penetration_rate FROM district_stats "
            "WHERE district_id = %s", parcel["district_id"])
        ev_score = min(100, (ev_rate["ev_penetration_rate"] or 0) * 200)  # 50% = 100分
        scores["ev_penetration"] = round(ev_score, 1)

        # 加权总分
        total = sum(
            scores[k] * self.SITE_WEIGHTS[k]
            for k in self.SITE_WEIGHTS
        )
        scores["total_score"] = round(total, 1)

        return scores

    # ---- 2. ROI 计算 ----

    def calculate_roi(self, site_plan):
        """计算建站投资回报率"""
        # 建设成本
        fast_chargers = site_plan["fast_charger_count"]
        slow_chargers = site_plan["slow_charger_count"]

        build_costs = {
            "land_lease": site_plan["area_sqm"] * site_plan["lease_price_per_sqm"] * 12 * 10,  # 10 年租金
            "fast_charger_equipment": fast_chargers * 80000,     # 8万/台
            "slow_charger_equipment": slow_chargers * 15000,     # 1.5万/台
            "civil_engineering": (fast_chargers + slow_chargers) * 20000,  # 土建 2万/桩位
            "grid_connection": min(500000, fast_chargers * 60 * 5000),     # 并网费
            "transformer": 300000 if fast_chargers > 5 else 150000,       # 变压器
            "other": 200000,  # 消防、监控、标识等
        }
        total_build_cost = sum(build_costs.values())

        # 年收入预估
        daily_sessions_fast = fast_chargers * 8    # 快充日均 8 次
        daily_sessions_slow = slow_chargers * 3    # 慢充日均 3 次
        avg_kwh_fast = 40                          # 快充均次 40 度
        avg_kwh_slow = 30                          # 慢充均次 30 度
        service_fee_per_kwh = 0.8                  # 服务费 0.8 元/度

        annual_revenue = (
            daily_sessions_fast * avg_kwh_fast * service_fee_per_kwh +
            daily_sessions_slow * avg_kwh_slow * service_fee_per_kwh
        ) * 365

        # 年运营成本
        electricity_cost_per_kwh = 0.5             # 购电成本 0.5 元/度
        annual_electricity = (
            daily_sessions_fast * avg_kwh_fast +
            daily_sessions_slow * avg_kwh_slow
        ) * 365 * electricity_cost_per_kwh

        annual_operation = (
            annual_electricity +                    # 电费
            120000 +                                # 场地维护
            60000 +                                 # 设备维护
            100000                                  # 人工+保险
        )

        annual_net_profit = annual_revenue - annual_operation
        payback_years = total_build_cost / max(annual_net_profit, 1)
        roi = (annual_net_profit * 10 - total_build_cost) / total_build_cost  # 10 年 ROI

        return {
            "total_build_cost": round(total_build_cost, 0),
            "build_cost_breakdown": {k: round(v, 0) for k, v in build_costs.items()},
            "annual_revenue": round(annual_revenue, 0),
            "annual_operation_cost": round(annual_operation, 0),
            "annual_net_profit": round(annual_net_profit, 0),
            "payback_years": round(payback_years, 1),
            "roi_10year": round(roi, 3),
            "recommended": roi > 0.5 and payback_years < 5
        }

    # ---- 3. 合规检查 ----

    def check_zoning_compliance(self, parcel_id):
        """用地合规性检查"""
        parcel = self.db.get_parcel(parcel_id)
        checks = []

        # 用地性质
        allowed_zones = ["commercial", "industrial", "mixed_use"]
        zone_ok = parcel["zoning_type"] in allowed_zones
        checks.append({
            "item": "用地性质",
            "status": "pass" if zone_ok else "fail",
            "detail": f"当前用地类型: {parcel['zoning_type']}"
        })

        # 消防距离（距居民楼 >= 50 米）
        residential_dist = self._nearest_residential_distance(parcel)
        fire_ok = residential_dist >= 50
        checks.append({
            "item": "消防距离",
            "status": "pass" if fire_ok else "fail",
            "detail": f"距最近居民楼: {residential_dist}米" + ("" if fire_ok else "（需 >= 50米）")
        })

        # 环评要求
        eia_required = parcel["area_sqm"] > 2000  # 超过 2000 平方米需环评
        checks.append({
            "item": "环评要求",
            "status": "info",
            "detail": "需要环评报告" if eia_required else "无需环评"
        })

        # 面积要求
        min_area = (parcel.get("planned_chargers", 10)) * 25  # 每桩位至少 25 平方米
        area_ok = parcel["area_sqm"] >= min_area
        checks.append({
            "item": "面积要求",
            "status": "pass" if area_ok else "fail",
            "detail": f"面积{parcel['area_sqm']}sqm" + (f"（需 >= {min_area}sqm）" if not area_ok else "")
        })

        all_pass = all(c["status"] != "fail" for c in checks)
        return {"compliant": all_pass, "checks": checks}

    def _nearest_residential_distance(self, parcel):
        """计算距最近居民楼的距离（米）"""
        return self.gis_service.nearest_residential(parcel["lat"], parcel["lng"])

    # ---- 4. 并网申请 ----

    def submit_grid_connection_application(self, station_id):
        """提交并网申请"""
        station = self.db.get_station(station_id)

        # 获取电网信息
        grid_info = self.grid_api.query_nearest_substation(
            station["lat"], station["lng"])

        # 生成申请材料
        application = {
            "station_id": station_id,
            "station_name": station["name"],
            "station_address": station["address"],
            "requested_capacity_kw": station["planned_capacity_kw"],
            "nearest_substation": grid_info["substation_id"],
            "distance_to_substation_km": grid_info["distance_km"],
            "application_type": "new_connection",
            "applicant": station["operator_name"],
            "documents": [
                "营业执照复印件",
                "用地证明",
                "消防审查意见",
                "环评报告（如需）",
                "电气设计方案",
            ],
            "status": "submitted",
            "submitted_at": now()
        }

        # 提交到电力公司
        result = self.grid_api.submit_application(application)

        self.db.insert("grid_applications", {
            **application,
            "grid_ref": result.get("reference_number"),
            "estimated_completion": result.get("estimated_date"),
        })

        return {
            "status": "submitted",
            "reference_number": result.get("reference_number"),
            "estimated_days": result.get("estimated_days", 60)
        }

    # ---- 5. 站点设计模板 ----

    def generate_station_design(self, site_plan):
        """生成站点设计方案"""
        area = site_plan["area_sqm"]
        total_chargers = site_plan.get("planned_chargers",
            max(4, int(area / 40)))  # 每 40 平方米一个桩位

        # 快慢充比例（基于选址特征调整）
        if site_plan.get("location_type") == "highway":
            fast_ratio = 0.80  # 高速服务区：80% 快充
        elif site_plan.get("location_type") == "shopping_mall":
            fast_ratio = 0.40  # 商场：40% 快充（用户停留时间长）
        elif site_plan.get("location_type") == "residential":
            fast_ratio = 0.20  # 住宅区：20% 快充
        else:
            fast_ratio = 0.50  # 默认 50%

        fast_count = max(2, int(total_chargers * fast_ratio))
        slow_count = total_chargers - fast_count

        # 功率配置
        fast_power_kw = 60   # 快充 60kW
        slow_power_kw = 7    # 慢充 7kW
        total_power_kw = fast_count * fast_power_kw + slow_count * slow_power_kw

        # 变压器容量（1.2 倍余量）
        transformer_kva = int(total_power_kw * 1.2)

        design = {
            "site_plan": site_plan["parcel_id"],
            "layout": {
                "fast_chargers": fast_count,
                "slow_chargers": slow_count,
                "fast_power_kw": fast_power_kw,
                "slow_power_kw": slow_power_kw,
                "total_power_kw": total_power_kw,
                "transformer_kva": transformer_kva,
            },
            "facilities": {
                "parking_spaces": total_chargers + 2,  # 充电桩位 + 2 个备用
                "canopy": True,                          # 雨棚
                "cctv_cameras": max(4, total_chargers // 4),
                "rest_area": area > 500,                  # 超过 500 平方米设休息区
                "vending_machine": area > 300,
                "restroom": area > 800,
            },
            "safety": {
                "fire_extinguishers": max(4, total_chargers // 4),
                "emergency_stop_buttons": max(2, total_chargers // 6),
                "ventilation": True,
                "insulation_monitoring": True,
            }
        }

        return design
```

## 用户充电行为分析完整实现（深度版）

```python
class ChargingBehaviorAnalysisService:
    """充电行为分析：聚类 -> 时段偏好 -> 忠诚度 -> 流失预测 -> 个性化推荐"""

    # ---- 1. 充电模式聚类 ----

    BEHAVIOR_PATTERNS = {
        "commuter": {       # 通勤型
            "typical_times": ["07:00-09:00", "18:00-21:00"],
            "weekly_frequency": "5-10",
            "avg_kwh": 20,
            "preferred_type": "fast",
        },
        "weekend": {        # 周末型
            "typical_times": ["10:00-18:00"],
            "weekly_frequency": "1-3",
            "avg_kwh": 35,
            "preferred_type": "mixed",
        },
        "long_trip": {      # 长途型
            "typical_times": ["06:00-22:00"],
            "weekly_frequency": "0-2",
            "avg_kwh": 50,
            "preferred_type": "fast",
        },
    }

    def classify_user_pattern(self, user_id):
        """分类用户充电模式"""
        sessions = self.db.query(
            "SELECT * FROM charging_sessions WHERE user_id = %s "
            "AND status = 'completed' AND end_time >= %s "
            "ORDER BY start_time",
            user_id, now() - timedelta(days=90))

        if len(sessions) < 3:
            return {"pattern": "insufficient_data", "confidence": 0}

        # 提取特征
        features = self._extract_behavior_features(sessions)

        # 聚类分类
        scores = {}
        for pattern_name, pattern_def in self.BEHAVIOR_PATTERNS.items():
            score = self._calculate_pattern_match(features, pattern_def)
            scores[pattern_name] = score

        best_pattern = max(scores, key=scores.get)

        return {
            "user_id": user_id,
            "pattern": best_pattern,
            "confidence": round(scores[best_pattern], 3),
            "scores": scores,
            "features": features
        }

    def _extract_behavior_features(self, sessions):
        """提取行为特征"""
        hours = [s["start_time"].hour for s in sessions]
        weekdays = [s["start_time"].weekday() for s in sessions]
        kwh_values = [s["kwh_consumed"] for s in sessions]

        # 工作日 vs 周末比例
        weekday_count = sum(1 for d in weekdays if d < 5)
        weekend_count = len(weekdays) - weekday_count
        weekday_ratio = weekday_count / max(len(weekdays), 1)

        # 早晚高峰比例
        morning_peak = sum(1 for h in hours if 7 <= h <= 9)
        evening_peak = sum(1 for h in hours if 18 <= h <= 21)
        peak_ratio = (morning_peak + evening_peak) / max(len(hours), 1)

        return {
            "avg_kwh": round(sum(kwh_values) / len(kwh_values), 1),
            "weekday_ratio": round(weekday_ratio, 3),
            "peak_ratio": round(peak_ratio, 3),
            "weekly_frequency": round(len(sessions) / 12.86, 1),  # 90 天约 12.86 周
            "fast_charge_ratio": round(
                sum(1 for s in sessions if s["charge_type"] == "fast") / len(sessions), 3),
        }

    def _calculate_pattern_match(self, features, pattern_def):
        """计算特征与模式定义的匹配度"""
        score = 0

        if pattern_def == self.BEHAVIOR_PATTERNS["commuter"]:
            score += features["peak_ratio"] * 0.4
            score += features["weekday_ratio"] * 0.3
            if features["avg_kwh"] < 30:
                score += 0.2
            if features["fast_charge_ratio"] > 0.5:
                score += 0.1

        elif pattern_def == self.BEHAVIOR_PATTERNS["weekend"]:
            score += (1 - features["weekday_ratio"]) * 0.4
            if 25 < features["avg_kwh"] < 45:
                score += 0.3
            if features["weekly_frequency"] < 3:
                score += 0.2
            score += (1 - features["peak_ratio"]) * 0.1

        elif pattern_def == self.BEHAVIOR_PATTERNS["long_trip"]:
            if features["avg_kwh"] > 40:
                score += 0.4
            if features["fast_charge_ratio"] > 0.7:
                score += 0.3
            if features["weekly_frequency"] < 2:
                score += 0.2
            score += (1 - features["peak_ratio"]) * 0.1

        return min(1.0, score)

    # ---- 2. 偏好时段分析 ----

    def analyze_preferred_timeslots(self, user_id):
        """分析用户偏好的充电时段"""
        sessions = self.db.query(
            "SELECT * FROM charging_sessions WHERE user_id = %s "
            "AND status = 'completed' AND end_time >= %s",
            user_id, now() - timedelta(days=90))

        # 按时段统计
        timeslot_counts = {}
        for s in sessions:
            hour = s["start_time"].hour
            slot = self._hour_to_timeslot(hour)
            timeslot_counts[slot] = timeslot_counts.get(slot, 0) + 1

        # 排序
        sorted_slots = sorted(timeslot_counts.items(), key=lambda x: x[1], reverse=True)

        return {
            "user_id": user_id,
            "preferred_slots": sorted_slots[:3],
            "distribution": timeslot_counts
        }

    def _hour_to_timeslot(self, hour):
        """小时转为时段"""
        if 6 <= hour < 9:
            return "早高峰(6-9)"
        elif 9 <= hour < 12:
            return "上午(9-12)"
        elif 12 <= hour < 14:
            return "午间(12-14)"
        elif 14 <= hour < 18:
            return "下午(14-18)"
        elif 18 <= hour < 21:
            return "晚高峰(18-21)"
        elif 21 <= hour < 24:
            return "夜间(21-24)"
        else:
            return "凌晨(0-6)"

    # ---- 3. 站点忠诚度追踪 ----

    def track_station_loyalty(self, user_id):
        """追踪用户站点忠诚度"""
        sessions = self.db.query(
            "SELECT station_id, COUNT(*) as visit_count, "
            "MAX(start_time) as last_visit "
            "FROM charging_sessions WHERE user_id = %s "
            "AND status = 'completed' AND end_time >= %s "
            "GROUP BY station_id ORDER BY visit_count DESC",
            user_id, now() - timedelta(days=90))

        total_visits = sum(s["visit_count"] for s in sessions)
        if total_visits == 0:
            return {"loyalty_score": 0}

        # 忠诚度 = 最常去站点占比
        top_station = sessions[0] if sessions else None
        top_ratio = top_station["visit_count"] / total_visits if top_station else 0

        # 忠诚度等级
        if top_ratio >= 0.7:
            loyalty_level = "high"
        elif top_ratio >= 0.4:
            loyalty_level = "medium"
        else:
            loyalty_level = "low"

        return {
            "user_id": user_id,
            "total_visits": total_visits,
            "unique_stations": len(sessions),
            "top_station_id": top_station["station_id"] if top_station else None,
            "top_station_visits": top_station["visit_count"] if top_station else 0,
            "loyalty_score": round(top_ratio, 3),
            "loyalty_level": loyalty_level,
            "station_distribution": sessions[:5]
        }

    # ---- 4. 流失预测 ----

    def predict_churn_risk(self, user_id):
        """预测用户流失风险"""
        last_session = self.db.query_one(
            "SELECT * FROM charging_sessions WHERE user_id = %s "
            "AND status = 'completed' ORDER BY end_time DESC LIMIT 1",
            user_id)

        if not last_session:
            return {"churn_risk": "high", "reason": "从未充电", "days_since_last": None}

        days_since_last = (now() - last_session["end_time"]).days

        # 基于充电间隔判断流失风险
        avg_interval = self._get_avg_charging_interval(user_id)

        if days_since_last > avg_interval * 3:
            risk = "high"
        elif days_since_last > avg_interval * 2:
            risk = "medium"
        elif days_since_last > avg_interval * 1.5:
            risk = "low"
        else:
            risk = "none"

        return {
            "user_id": user_id,
            "days_since_last_charge": days_since_last,
            "avg_interval_days": round(avg_interval, 1),
            "churn_risk": risk,
            "last_station_id": last_session["station_id"],
            "recommended_action": self._get_churn_action(risk, days_since_last)
        }

    def _get_avg_charging_interval(self, user_id):
        """获取平均充电间隔天数"""
        sessions = self.db.query(
            "SELECT start_time FROM charging_sessions WHERE user_id = %s "
            "AND status = 'completed' ORDER BY start_time LIMIT 20",
            user_id)

        if len(sessions) < 2:
            return 30  # 默认 30 天

        intervals = []
        for i in range(1, len(sessions)):
            delta = (sessions[i]["start_time"] - sessions[i-1]["start_time"]).days
            intervals.append(delta)

        return sum(intervals) / len(intervals)

    def _get_churn_action(self, risk, days_since_last):
        """获取流失应对措施"""
        if risk == "high":
            return f"已 {days_since_last} 天未充电，发送优惠券+个性化推荐"
        elif risk == "medium":
            return f"已 {days_since_last} 天未充电，推送附近站点优惠"
        elif risk == "low":
            return "轻度流失风险，常规运营触达"
        return "正常用户"

    # ---- 5. 个性化推荐 ----

    def recommend_stations(self, user_id, current_lat=None, current_lng=None):
        """个性化推荐充电站"""
        behavior = self.classify_user_pattern(user_id)
        loyalty = self.track_station_loyalty(user_id)

        # 获取候选站点
        if current_lat and current_lng:
            candidates = self.db.query(
                "SELECT *, ST_Distance_Sphere(location, ST_Point(%s, %s)) as distance_m "
                "FROM charging_stations WHERE status = 'active' "
                "HAVING distance_m <= 10000 ORDER BY distance_m LIMIT 20",
                current_lng, current_lat)
        else:
            candidates = self.db.query(
                "SELECT * FROM charging_stations WHERE status = 'active' "
                "AND city_id = (SELECT city_id FROM users WHERE user_id = %s) "
                "LIMIT 20", user_id)

        # 评分排序
        scored = []
        for station in candidates:
            score = 0

            # 距离分（越近越好）
            distance_m = station.get("distance_m", 5000)
            score += max(0, 50 - distance_m / 200)  # 5km 内最高 50 分

            # 忠诚度加分
            if station["station_id"] == loyalty.get("top_station_id"):
                score += 20

            # 空闲桩位加分
            available = self.redis.get(f"station:{station['station_id']}:available")
            if available and int(available) > 0:
                score += 15

            # 优惠加分
            has_discount = self.redis.get(f"station:{station['station_id']}:discount")
            if has_discount:
                score += 10

            # 行为匹配加分
            pattern = behavior.get("pattern", "")
            if pattern == "commuter" and station.get("has_fast_charge"):
                score += 5

            scored.append({"station": station, "score": round(score, 1)})

        scored.sort(key=lambda x: x["score"], reverse=True)
        return scored[:5]
```

## 充电网络扩展规划完整实现

```python
class ChargingNetworkExpansionPlanner:
    """充电网络扩展规划：覆盖缺口 -> 需求预测 -> 优先排序 -> 预算优化 -> 季度计划"""

    # ---- 1. 覆盖缺口检测 ----

    def detect_coverage_gaps(self, city_id, max_distance_km=5):
        """检测充电网络覆盖缺口（距最近站点 > 5km 的区域）"""
        stations = self.db.query(
            "SELECT * FROM charging_stations WHERE city_id = %s AND status = 'active'",
            city_id)

        population_grids = self.demographics_service.get_population_grids(city_id)

        gaps = []
        for grid in population_grids:
            min_distance = float('inf')
            nearest_station = None
            for station in stations:
                dist = self.haversine(
                    grid["center_lat"], grid["center_lng"],
                    station["lat"], station["lng"])
                if dist < min_distance:
                    min_distance = dist
                    nearest_station = station

            if min_distance > max_distance_km:
                gaps.append({
                    "grid_id": grid["grid_id"],
                    "center_lat": grid["center_lat"],
                    "center_lng": grid["center_lng"],
                    "population": grid["population"],
                    "ev_estimated": int(grid["population"] * grid.get("ev_penetration", 0.05)),
                    "distance_to_nearest_km": round(min_distance, 1),
                    "nearest_station_id": nearest_station["station_id"] if nearest_station else None
                })

        return sorted(gaps, key=lambda x: x["population"], reverse=True)

    # ---- 2. 需求预测（按区域） ----

    def forecast_demand_by_region(self, city_id, horizon_months=12):
        """按区域预测充电需求"""
        regions = self.db.query(
            "SELECT * FROM city_regions WHERE city_id = %s", city_id)

        forecasts = []
        for region in regions:
            historical = self.db.query(
                "SELECT DATE_TRUNC('month', start_time) as month, "
                "COUNT(*) as sessions, SUM(kwh_consumed) as total_kwh "
                "FROM charging_sessions cs "
                "JOIN charging_stations st ON cs.station_id = st.station_id "
                "WHERE st.region_id = %s AND cs.start_time >= %s "
                "GROUP BY month ORDER BY month",
                region["region_id"],
                now() - timedelta(days=365))

            if len(historical) < 3:
                base_demand = region["population"] * region.get("ev_penetration", 0.05) * 2
                growth_rate = 0.10
            else:
                recent = historical[-3:]
                base_demand = sum(h["sessions"] for h in recent) / 3
                if len(historical) >= 6:
                    old_avg = sum(h["sessions"] for h in historical[:3]) / 3
                    new_avg = sum(h["sessions"] for h in historical[-3:]) / 3
                    growth_rate = (new_avg / max(old_avg, 1) - 1) / 3
                else:
                    growth_rate = 0.05

            predicted = []
            for m in range(1, horizon_months + 1):
                demand = base_demand * (1 + growth_rate) ** m
                predicted.append({
                    "month_offset": m,
                    "predicted_sessions": round(demand),
                    "predicted_kwh": round(demand * 35)
                })

            forecasts.append({
                "region_id": region["region_id"],
                "region_name": region["name"],
                "current_monthly_sessions": round(base_demand),
                "monthly_growth_rate": round(growth_rate, 4),
                "forecast": predicted
            })

        return forecasts

    # ---- 3. 优先排序 ----

    def rank_expansion_priorities(self, city_id, budget=None):
        """扩展优先级排序（需求 x 缺口距离）"""
        gaps = self.detect_coverage_gaps(city_id)
        forecasts = self.forecast_demand_by_region(city_id)

        demand_map = {}
        for f in forecasts:
            demand_map[f["region_id"]] = f["current_monthly_sessions"]

        priorities = []
        for gap in gaps:
            region = self._find_region(gap["center_lat"], gap["center_lng"], city_id)
            regional_demand = demand_map.get(region["region_id"], 0) if region else 0

            gap_distance = gap["distance_to_nearest_km"]
            population_weight = min(gap["population"] / 50000, 3)

            priority_score = (
                regional_demand * 0.3 +
                gap_distance * 10 * 0.4 +
                population_weight * 100 * 0.3
            )

            estimated_cost = self._estimate_station_cost(gap)

            priorities.append({
                "gap_id": gap["grid_id"],
                "center_lat": gap["center_lat"],
                "center_lng": gap["center_lng"],
                "population": gap["population"],
                "ev_estimated": gap["ev_estimated"],
                "gap_distance_km": gap_distance,
                "regional_demand": regional_demand,
                "priority_score": round(priority_score, 1),
                "estimated_cost": round(estimated_cost, 0),
            })

        priorities.sort(key=lambda x: x["priority_score"], reverse=True)

        if budget:
            selected = []
            remaining = budget
            for p in priorities:
                if p["estimated_cost"] <= remaining:
                    selected.append(p)
                    remaining -= p["estimated_cost"]
            return selected

        return priorities

    def _find_region(self, lat, lng, city_id):
        """根据坐标查找所属区域"""
        return self.db.query_one(
            "SELECT * FROM city_regions WHERE city_id = %s "
            "AND ST_Contains(boundary, ST_Point(%s, %s))",
            city_id, lng, lat)

    def _estimate_station_cost(self, gap):
        """估算建站成本"""
        base_cost = 800000
        charger_count = max(4, int(gap["ev_estimated"] / 50))
        cost = base_cost + charger_count * 40000
        return cost

    # ---- 4. 预算优化 ----

    def optimize_budget(self, city_id, annual_budget):
        """在预算内最大化覆盖率"""
        priorities = self.rank_expansion_priorities(city_id)

        selected = []
        remaining_budget = annual_budget
        covered_population = 0

        for p in priorities:
            if p["estimated_cost"] <= remaining_budget:
                selected.append(p)
                remaining_budget -= p["estimated_cost"]
                covered_population += p["population"]

        total_gap_population = sum(p["population"] for p in priorities)

        return {
            "annual_budget": annual_budget,
            "budget_used": round(annual_budget - remaining_budget, 0),
            "budget_remaining": round(remaining_budget, 0),
            "stations_planned": len(selected),
            "covered_population": covered_population,
            "coverage_rate": round(covered_population / max(total_gap_population, 1), 3),
            "selected_sites": selected
        }

    # ---- 5. 季度扩展计划生成 ----

    def generate_quarterly_plan(self, city_id, quarter, annual_budget):
        """生成季度扩展计划"""
        quarterly_budget = annual_budget / 4

        priorities = self.rank_expansion_priorities(city_id)

        selected = []
        remaining = quarterly_budget
        for p in priorities:
            if p["estimated_cost"] <= remaining and p not in self._get_planned_sites(city_id):
                selected.append(p)
                remaining -= p["estimated_cost"]

        projects = []
        for i, site in enumerate(selected):
            start_week = i * 4 + 1
            projects.append({
                "project_id": f"P-{city_id}-{quarter}-{'%03d' % (i+1)}",
                "site": site,
                "timeline": {
                    "design_week": start_week,
                    "permit_week": start_week + 2,
                    "construction_week": start_week + 4,
                    "commissioning_week": start_week + 10,
                    "target_open_week": start_week + 12,
                },
                "estimated_cost": site["estimated_cost"],
                "status": "planned"
            })

        return {
            "city_id": city_id,
            "quarter": quarter,
            "budget": round(quarterly_budget, 0),
            "budget_used": round(quarterly_budget - remaining, 0),
            "project_count": len(projects),
            "projects": projects
        }

    def _get_planned_sites(self, city_id):
        """获取已纳入计划的站点"""
        existing = self.db.query(
            "SELECT center_lat, center_lng FROM expansion_projects "
            "WHERE city_id = %s AND status IN ('planned', 'in_progress')",
            city_id)
        return existing
```

## 异常场景补充

### 场景：选址规划数据不足

```
触发：新进入城市缺乏交通流量、人口密度等基础数据 -> 无法评估选址
检测：
  1. 选址评分中任一维度数据缺失 -> 数据不足标记
  2. 可用维度 < 3 个 -> 无法生成可靠评分
处理：
  1. 降级评估：使用可用维度计算部分评分，缺失维度给默认中位值
  2. 标记为"低置信度"选址建议，需人工复核
  3. 同步启动数据采集：接入第三方地图API获取交通数据
  4. 数据补全后重新评分
预防：多数据源冗余 + 数据质量检查 + 降级评估策略
```

### 场景：扩展计划超出预算

```
触发：所有高优先级站点总造价 > 年度预算 -> 无法全部建设
检测：
  1. 季度规划总造价 > 季度预算 -> 超预算
  2. 预算利用率 < 60%（选站过于保守）-> 预算浪费
处理：
  1. 超预算：按优先级截断，保留最高优先级站点
  2. 对截断的站点评估"小型站"替代方案（减少桩数、降低功率）
  3. 申请追加预算（附 ROI 数据）
  4. 预算利用率低：适当增加中优先级站点
预防：预算弹性规划 + 分期建设选项 + 成本优化方案

## 充电桩远程运维完整实现

```python
class ChargingStationRemoteOps:
    """充电桩远程运维：远程重启 + 诊断 + 固件更新"""

    def remote_diagnosis(self, station_id, charger_id):
        """远程诊断"""
        charger = self.db.get_charger(station_id, charger_id)

        diagnostics = {
            "charger_id": charger_id,
            "station_id": station_id,
            "timestamp": now().isoformat(),
        }

        # 1. 硬件状态
        diagnostics["hardware"] = {
            "cpu_usage_pct": self._read_telemetry(charger_id, "cpu_usage"),
            "memory_usage_pct": self._read_telemetry(charger_id, "memory_usage"),
            "temperature_c": self._read_telemetry(charger_id, "temperature"),
            "uptime_hours": self._read_telemetry(charger_id, "uptime"),
        }

        # 2. 通信状态
        diagnostics["communication"] = {
            "mqtt_connected": self._check_mqtt_connection(charger_id),
            "last_heartbeat": self.redis.get(f"heartbeat:{charger_id}"),
            "signal_strength_dbm": self._read_telemetry(charger_id, "signal_strength"),
        }

        # 3. 充电模块状态
        diagnostics["charging_module"] = {
            "input_voltage_v": self._read_telemetry(charger_id, "input_voltage"),
            "output_voltage_v": self._read_telemetry(charger_id, "output_voltage"),
            "output_current_a": self._read_telemetry(charger_id, "output_current"),
            "power_factor": self._read_telemetry(charger_id, "power_factor"),
        }

        # 4. 故障码
        diagnostics["fault_codes"] = self._read_fault_codes(charger_id)

        # 5. 自动诊断建议
        diagnostics["recommendations"] = self._generate_recommendations(diagnostics)

        return diagnostics

    def remote_restart(self, station_id, charger_id, restart_type="soft"):
        """远程重启"""
        if restart_type == "soft":
            # 软重启：重启充电控制程序
            self.mqtt_client.publish(
                f"charger/{station_id}/{charger_id}/command",
                json.dumps({"action": "restart", "type": "soft"}))
        elif restart_type == "hard":
            # 硬重启：断电重启（需要确认无正在进行的充电会话）
            active_session = self.db.query_one(
                "SELECT * FROM charging_sessions "
                "WHERE charger_id = %s AND status = 'charging'", charger_id)
            if active_session:
                return {"status": "rejected", "reason": "有正在进行的充电会话"}

            self.mqtt_client.publish(
                f"charger/{station_id}/{charger_id}/command",
                json.dumps({"action": "restart", "type": "hard"}))

        self.db.insert("remote_operations_log", {
            "operation_id": str(uuid4()),
            "station_id": station_id,
            "charger_id": charger_id,
            "operation": f"restart_{restart_type}",
            "operator": "system",
            "executed_at": now()
        })

        return {"status": "sent", "restart_type": restart_type}

    def _generate_recommendations(self, diagnostics):
        """生成诊断建议"""
        recs = []
        hw = diagnostics.get("hardware", {})

        if hw.get("temperature_c", 0) > 70:
            recs.append({"priority": "high", "action": "检查散热系统",
                        "reason": f"温度 {hw['temperature_c']}°C 超标"})
        if hw.get("cpu_usage_pct", 0) > 90:
            recs.append({"priority": "medium", "action": "建议软重启",
                        "reason": f"CPU 使用率 {hw['cpu_usage_pct']}%"})
        if not diagnostics.get("communication", {}).get("mqtt_connected"):
            recs.append({"priority": "high", "action": "检查网络连接",
                        "reason": "MQTT 断开"})

        fault_codes = diagnostics.get("fault_codes", [])
        for fc in fault_codes:
            recs.append({"priority": "critical", "action": f"处理故障码 {fc['code']}",
                        "reason": fc.get("description", "未知故障")})

        return recs
```

## 充电站能源管理

```python
class StationEnergyManager:
    """充电站能源管理：负荷均衡 + 峰谷电价 + 光伏集成"""

    PEAK_HOURS = [(8, 11), (17, 21)]  # 峰电时段
    VALLEY_HOURS = [(23, 7)]  # 谷电时段

    def optimize_load_distribution(self, station_id):
        """优化负荷分配"""
        # 1. 获取当前负荷
        chargers = self.db.query(
            "SELECT * FROM chargers WHERE station_id = %s AND status = 'active'",
            station_id)

        total_power_kw = 0
        charging_sessions = []
        for c in chargers:
            session = self.db.query_one(
                "SELECT * FROM charging_sessions "
                "WHERE charger_id = %s AND status = 'charging'", c["id"])
            if session:
                power = session["current_power_kw"]
                total_power_kw += power
                charging_sessions.append({"charger_id": c["id"], "power": power,
                    "battery_pct": session["battery_percentage"], "requested_power": session["requested_power_kw"]})

        # 2. 站点总容量限制
        station = self.db.get_station(station_id)
        max_capacity_kw = station["max_power_capacity_kw"]

        # 3. 如果超容量 → 降功率分配
        if total_power_kw > max_capacity_kw * 0.9:
            # 优先保障低电量车辆
            charging_sessions.sort(key=lambda x: x["battery_pct"])
            allocated = self._allocate_power(charging_sessions, max_capacity_kw * 0.9)

            for session in allocated:
                self._set_charger_power(session["charger_id"], session["allocated_power_kw"])

            return {"action": "load_balanced", "total_demand_kw": total_power_kw,
                    "capacity_kw": max_capacity_kw, "sessions_adjusted": len(allocated)}

        return {"action": "no_adjustment_needed", "total_kw": total_power_kw,
                "capacity_kw": max_capacity_kw}

    def calculate_electricity_cost(self, station_id, kwh, timestamp=None):
        """计算电费（峰谷电价）"""
        ts = timestamp or now()
        hour = ts.hour

        # 判断峰谷时段
        rate = "normal"
        for start, end in self.PEAK_HOURS:
            if start <= hour < end:
                rate = "peak"
                break
        for start, end in self.VALLEY_HOURS:
            if start <= hour < end or (start > end and (hour >= start or hour < end)):
                rate = "valley"
                break

        rates = {"peak": 1.2, "normal": 0.8, "valley": 0.4}  # 元/kWh
        cost = kwh * rates[rate]

        return {"kwh": kwh, "rate_type": rate, "rate_per_kwh": rates[rate],
                "total_cost_cny": round(cost, 2)}
```

## 异常场景补充

### 场景：远程重启后充电桩无响应

```
触发：远程硬重启 → 充电桩重启后未重新连接 → 完全失联
检测：
  1. 重启后 5 分钟无心跳 → 失联
  2. MQTT 连接未恢复 → 通信故障
处理：
  1. 派遣现场运维人员
  2. 尝试通过网络交换机重启端口
  3. 标记充电桩为故障
预防：重启前确认网络稳定 + 重启超时检测 + 现场运维 SLA
```

### 场景：负荷均衡导致充电速度投诉

```
触发：站点超容量 → 降低充电功率 → 用户抱怨充电太慢
检测：
  1. 用户投诉充电速度 → 负荷均衡影响
  2. 平均充电功率 < 标称 50% → 过度降功率
处理：
  1. 优化分配策略（优先保障新接入车辆全功率）
  2. 通知用户当前站点繁忙
  3. 推荐附近空闲站点
预防：智能功率分配 + 用户通知 + 附近站点推荐
```

## 充电站站点选址优化完整实现

```python
class ChargingSiteSelectionService:
    """充电站选址优化：需求预测 + 成本分析 + 竞品分析"""

    def evaluate_site(self, candidate_lat, candidate_lng, radius_km=3):
        """评估候选站点"""
        # 1. 需求预测（周边 EV 密度 + 充电缺口）
        ev_density = self._estimate_ev_density(candidate_lat, candidate_lng, radius_km)
        charging_gap = self._estimate_charging_gap(candidate_lat, candidate_lng, radius_km)

        # 2. 交通流量
        traffic_flow = self._get_traffic_flow(candidate_lat, candidate_lng)

        # 3. 竞品分析
        competitors = self._find_nearby_stations(candidate_lat, candidate_lng, radius_km)
        competition_score = self._calculate_competition(competitors, radius_km)

        # 4. 土地与建设成本
        land_cost = self._estimate_land_cost(candidate_lat, candidate_lng)
        construction_cost = self._estimate_construction_cost(candidate_lat, candidate_lng)

        # 5. 电网接入成本
        grid_connection = self._estimate_grid_connection(candidate_lat, candidate_lng)

        # 6. 预估营收
        estimated_daily_sessions = charging_gap * 0.6  # 60% 的缺口可被捕获
        revenue_per_session = 30  # 平均 30 元/次
        daily_revenue = estimated_daily_sessions * revenue_per_session

        # 7. ROI 计算
        total_investment = land_cost + construction_cost + grid_connection
        annual_revenue = daily_revenue * 365
        annual_opex = total_investment * 0.05  # 5% 运维成本
        annual_profit = annual_revenue - annual_opex
        roi_years = total_investment / max(annual_profit, 1)

        return {
            "location": {"lat": candidate_lat, "lng": candidate_lng},
            "market": {
                "ev_density": ev_density,
                "charging_gap": charging_gap,
                "estimated_daily_sessions": round(estimated_daily_sessions, 0),
                "competitors_within_radius": len(competitors),
                "competition_score": round(competition_score, 3),
            },
            "financial": {
                "land_cost": round(land_cost, 0),
                "construction_cost": round(construction_cost, 0),
                "grid_connection": round(grid_connection, 0),
                "total_investment": round(total_investment, 0),
                "daily_revenue": round(daily_revenue, 0),
                "annual_profit": round(annual_profit, 0),
                "roi_years": round(roi_years, 1),
            },
            "recommendation": "recommended" if roi_years < 3 and competition_score < 0.7 else
                             "consider" if roi_years < 5 else "not_recommended"
        }

    def _estimate_ev_density(self, lat, lng, radius_km):
        """估算周边 EV 密度"""
        # 基于车牌注册数据 + 充电历史数据
        registered_evs = self.db.query_one(
            "SELECT COUNT(*) as count FROM ev_registrations "
            "WHERE ST_DWithin(location, ST_MakePoint(%s, %s), %s)",
            lng, lat, radius_km / 111)["count"] or 0

        # 基于充电历史推算
        historical_sessions = self.db.query_one(
            "SELECT COUNT(*) as count FROM charging_sessions "
            "WHERE ST_DWithin(location, ST_MakePoint(%s, %s), %s) "
            "AND created_at > NOW() - INTERVAL 30 DAY",
            lng, lat, radius_km / 111)["count"] or 0

        return {"registered_evs": registered_evs,
                "monthly_sessions": historical_sessions}

    def _estimate_charging_gap(self, lat, lng, radius_km):
        """估算充电缺口（需求 - 供给）"""
        demand = self._estimate_daily_demand(lat, lng, radius_km)
        supply = self._estimate_daily_supply(lat, lng, radius_km)
        return max(0, demand - supply)

    def _calculate_competition(self, competitors, radius_km):
        """计算竞争分数"""
        if not competitors:
            return 0  # 无竞争

        # 基于距离和容量加权
        total_score = 0
        for c in competitors:
            distance = self._haversine(c["lat"], c["lng"])
            # 距离越近竞争越强
            proximity_score = max(0, 1 - distance / radius_km)
            # 容量越大竞争越强
            capacity_score = c["charger_count"] / 20
            total_score += proximity_score * capacity_score

        return min(1.0, total_score)
```

## 异常场景补充

### 场景：选址数据过时导致误判

```
触发：EV 注册数据是半年前的 → 当前 EV 数量已翻倍 → 选址评估低估需求
检测：
  1. 评估数据时间戳 > 3 个月 → 数据可能过时
  2. 实际充电量 > 预估 2 倍 → 数据不准
处理：
  1. 更新 EV 注册数据
  2. 重新评估选址
  3. 加入数据时效性标记
预防：数据定期更新 + 时效性标记 + 实时校准
```

### 场景：电网接入成本估算偏差大

```
触发：估算电网接入 50 万 → 实际需要 200 万（距离变电站远）→ ROI 严重偏差
检测：
  1. 实际电网接入成本 > 估算 3 倍 → 偏差大
  2. 项目预算超支 → 估算问题
处理：
  1. 请电力公司出具准确报价
  2. 修正选址评估模型
  3. 加入电网容量检查
预防：电力公司报价 + 电网容量检查 + 偏差安全系数
```

## 充电桩预约系统完整实现

```python
class ChargingReservationService:
    """充电预约：时段预约 + 排队 + 超时释放"""

    def create_reservation(self, user_id, station_id, charger_type,
                          start_time, estimated_duration_minutes=60):
        """创建充电预约"""
        # 1. 检查用户是否有进行中的预约
        active = self.db.query_one(
            "SELECT * FROM charging_reservations "
            "WHERE user_id = %s AND status IN ('confirmed', 'charging') "
            "AND expires_at > NOW()", user_id)
        if active:
            return {"status": "already_reserved", "reservation_id": active["id"]}

        # 2. 查找可用充电桩
        available = self.db.query(
            "SELECT * FROM chargers "
            "WHERE station_id = %s AND charger_type = %s "
            "AND status = 'available' "
            "AND id NOT IN ("
            "  SELECT charger_id FROM charging_reservations "
            "  WHERE status = 'confirmed' "
            "  AND start_time <= %s + INTERVAL %s MINUTE "
            "  AND end_time >= %s"
            ") LIMIT 1",
            station_id, charger_type,
            start_time, estimated_duration_minutes, start_time)

        if not available:
            # 无可用桩 → 加入排队
            queue_position = self._join_queue(user_id, station_id, charger_type, start_time)
            return {"status": "queued", "queue_position": queue_position}

        charger = available[0]

        # 3. 创建预约
        reservation_id = str(uuid4())
        end_time = start_time + timedelta(minutes=estimated_duration_minutes)

        self.db.insert("charging_reservations", {
            "reservation_id": reservation_id,
            "user_id": user_id,
            "station_id": station_id,
            "charger_id": charger["id"],
            "charger_type": charger_type,
            "start_time": start_time,
            "end_time": end_time,
            "status": "confirmed",
            "created_at": now()
        })

        # 4. 锁定充电桩
        self.db.update("chargers",
            {"status": "reserved"}, {"id": charger["id"]})

        # 5. 设置超时释放（预约开始后 15 分钟未到 → 自动取消）
        delay_seconds = (start_time - now()).total_seconds() + 15 * 60
        self.scheduler.schedule(delay_seconds,
            self._auto_cancel_no_show, reservation_id)

        # 6. 通知用户
        self.notification.send(user_id,
            f"预约成功: {station_id} {charger['id']}号桩, "
            f"时间: {start_time.strftime('%H:%M')}-{end_time.strftime('%H:%M')}")

        return {"reservation_id": reservation_id, "charger_id": charger["id"],
                "start_time": start_time.isoformat(),
                "end_time": end_time.isoformat()}

    def cancel_reservation(self, reservation_id, user_id, reason=None):
        """取消预约"""
        reservation = self.db.get_reservation(reservation_id)

        if reservation["user_id"] != user_id:
            raise PermissionDeniedError("非预约用户无法取消")

        if reservation["status"] not in ["confirmed", "queued"]:
            return {"status": "cannot_cancel", "current_status": reservation["status"]}

        # 1. 更新预约状态
        self.db.update("charging_reservations",
            {"status": "cancelled", "cancelled_at": now(), "cancel_reason": reason},
            {"reservation_id": reservation_id})

        # 2. 释放充电桩
        if reservation.get("charger_id"):
            self.db.update("chargers",
                {"status": "available"}, {"id": reservation["charger_id"]})

        # 3. 通知排队中的下一个用户
        if reservation["status"] == "confirmed":
            next_in_queue = self._get_next_in_queue(
                reservation["station_id"], reservation["charger_type"])
            if next_in_queue:
                self._promote_from_queue(next_in_queue, reservation)

        return {"status": "cancelled"}

    def _auto_cancel_no_show(self, reservation_id):
        """超时未到 → 自动取消"""
        reservation = self.db.get_reservation(reservation_id)

        if reservation["status"] != "confirmed":
            return  # 已取消或已开始充电

        # 标记为爽约
        self.db.update("charging_reservations",
            {"status": "no_show", "cancelled_at": now()},
            {"reservation_id": reservation_id})

        # 释放充电桩
        if reservation.get("charger_id"):
            self.db.update("chargers",
                {"status": "available"}, {"id": reservation["charger_id"]})

        # 记录爽约（影响信用分）
        self.db.insert("user_no_shows", {
            "user_id": reservation["user_id"],
            "reservation_id": reservation_id,
            "station_id": reservation["station_id"],
            "created_at": now()
        })

        # 通知排队用户
        next_in_queue = self._get_next_in_queue(
            reservation["station_id"], reservation["charger_type"])
        if next_in_queue:
            self._promote_from_queue(next_in_queue, reservation)

    def _join_queue(self, user_id, station_id, charger_type, preferred_time):
        """加入排队"""
        position = self.db.count("charging_reservations",
            station_id=station_id, charger_type=charger_type,
            status="queued") + 1

        self.db.insert("charging_reservations", {
            "reservation_id": str(uuid4()),
            "user_id": user_id,
            "station_id": station_id,
            "charger_type": charger_type,
            "start_time": preferred_time,
            "status": "queued",
            "queue_position": position,
            "created_at": now()
        })

        return position
```

## 异常场景补充

### 场景：预约超时释放后用户才到

```
触发：用户预约 10:00 → 10:15 未到自动取消 → 10:16 用户到达 → 桩已被他人占用
检测：
  1. 用户投诉预约被取消但已到达 → 超时边界问题
  2. 取消时间与用户到达时间差 < 2 分钟 → 边界情况
处理：
  1. 延长超时时间（15 分钟 → 20 分钟）
  2. 用户到达时如果桩仍可用 → 恢复预约
  3. 到达后桩被占用 → 优先排队
预防：延长超时 + 到达恢复 + 优先排队
```

### 场景：爽约用户反复预约

```
触发：用户多次爽约 → 充电桩资源浪费 → 其他用户无法预约
检测：
  1. 用户爽约次数 > 3 次/月 → 滥用预约
  2. 爽约率 > 20% → 预约制度失效
处理：
  1. 爽约 3 次后限制预约（7 天内不可预约）
  2. 预约需预付定金（爽约不退）
  3. 信用分系统
预防：爽约限制 + 预付定金 + 信用分
```

## 充电桩远程诊断与维护完整实现

```python
import time
import uuid
import threading
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from collections import defaultdict
from datetime import datetime, timedelta


class ChargerStatus(Enum):
    ONLINE = "online"
    CHARGING = "charging"
    OFFLINE = "offline"
    FAULT = "fault"
    MAINTENANCE = "maintenance"


class AnomalyType(Enum):
    OVERCURRENT = "overcurrent"
    OVERTEMPERATURE = "overtemperature"
    COMMUNICATION_FAILURE = "communication_failure"
    VOLTAGE_ABNORMAL = "voltage_abnormal"
    INSULATION_FAILURE = "insulation_failure"
    GROUND_FAULT = "ground_fault"


class MaintenancePriority(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"


class ResetResult(Enum):
    SUCCESS = "success"
    FAILED = "failed"
    TIMEOUT = "timeout"
    NOT_RESPONSIVE = "not_responsive"


@dataclass
class TelemetryData:
    """充电桩遥测数据"""
    charger_id: str
    voltage: float  # V
    current: float  # A
    temperature: float  # °C
    humidity: float  # %
    power: float  # kW
    energy_delivered: float  # kWh
    error_codes: List[str]
    status: ChargerStatus
    firmware_version: str
    uptime_seconds: int
    last_heartbeat: datetime
    timestamp: datetime = field(default_factory=datetime.now)


@dataclass
class AnomalyRecord:
    """异常记录"""
    anomaly_id: str
    charger_id: str
    anomaly_type: AnomalyType
    severity: MaintenancePriority
    description: str
    telemetry_snapshot: Dict
    detected_at: datetime
    acknowledged: bool = False
    resolved_at: Optional[datetime] = None


@dataclass
class MaintenanceTicket:
    """维护工单"""
    ticket_id: str
    charger_id: str
    anomaly_ids: List[str]
    priority: MaintenancePriority
    description: str
    assigned_technician: Optional[str] = None
    status: str = "open"  # open, in_progress, resolved, closed
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    resolution_notes: Optional[str] = None


@dataclass
class ResetCommand:
    """远程重置命令"""
    command_id: str
    charger_id: str
    reset_type: str  # soft_reset, hard_reset, firmware_update
    issued_by: str
    issued_at: datetime
    result: Optional[ResetResult] = None
    completed_at: Optional[datetime] = None
    response_data: Optional[Dict] = None


# 阈值配置
@dataclass
class AnomalyThresholds:
    max_current: float = 250.0  # A
    max_temperature: float = 85.0  # °C
    min_voltage: float = 200.0  # V
    max_voltage: float = 420.0  # V
    heartbeat_timeout_seconds: int = 120
    current_spike_ratio: float = 1.5  # 超过额定电流的倍数


class ChargerRemoteDiagnosticsService:
    """充电桩远程诊断与维护完整服务

    负责充电桩遥测数据采集、异常检测、维护工单管理及远程操作。
    """

    def __init__(self, thresholds: Optional[AnomalyThresholds] = None):
        # 充电桩遥测数据缓存 charger_id -> latest telemetry
        self._telemetry_cache: Dict[str, TelemetryData] = {}
        # 历史遥测数据 charger_id -> list of telemetry (最近N条)
        self._telemetry_history: Dict[str, List[TelemetryData]] = defaultdict(list)
        self._max_history_per_charger = 1000
        # 异常记录
        self._anomaly_records: Dict[str, AnomalyRecord] = {}
        # 维护工单
        self._maintenance_tickets: Dict[str, MaintenanceTicket] = {}
        # 远程重置命令记录
        self._reset_commands: Dict[str, ResetCommand] = {}
        # 阈值配置
        self._thresholds = thresholds or AnomalyThresholds()
        # 充电桩配置 charger_id -> config dict
        self._charger_configs: Dict[str, Dict] = {}
        # 充电桩额定电流表 charger_id -> rated_current
        self._rated_currents: Dict[str, float] = {}
        # 诊断锁
        self._diagnostics_lock = threading.Lock()
        # 异常回调
        self._anomaly_callbacks: List[callable] = []

    def collect_telemetry(self, telemetry: TelemetryData) -> Dict:
        """采集并存储充电桩遥测数据

        接收充电桩上报的遥测数据，存入缓存与历史记录，
        并立即执行异常检测。

        Args:
            telemetry: 遥测数据对象

        Returns:
            Dict: 包含采集确认及实时异常检测结果
        """
        charger_id = telemetry.charger_id

        # 存入缓存
        self._telemetry_cache[charger_id] = telemetry

        # 存入历史
        history = self._telemetry_history[charger_id]
        history.append(telemetry)
        if len(history) > self._max_history_per_charger:
            self._telemetry_history[charger_id] = history[-self._max_history_per_charger:]

        # 实时异常检测
        anomalies = self.detect_anomalies(charger_id)

        return {
            "charger_id": charger_id,
            "collected_at": telemetry.timestamp.isoformat(),
            "status": telemetry.status.value,
            "anomalies_detected": len(anomalies),
            "anomaly_types": [a.anomaly_type.value for a in anomalies],
        }

    def detect_anomalies(self, charger_id: str) -> List[AnomalyRecord]:
        """检测充电桩异常

        基于最新遥测数据，对电流、温度、电压、通信状态等进行
        多维度异常检测。

        Args:
            charger_id: 充电桩ID

        Returns:
            List[AnomalyRecord]: 检测到的异常列表
        """
        telemetry = self._telemetry_cache.get(charger_id)
        if telemetry is None:
            return []

        detected = []
        thresholds = self._thresholds
        now = datetime.now()

        # 过流检测
        rated_current = self._rated_currents.get(charger_id, 200.0)
        if telemetry.current > thresholds.max_current:
            anomaly = self._create_anomaly(
                charger_id=charger_id,
                anomaly_type=AnomalyType.OVERCURRENT,
                severity=MaintenancePriority.CRITICAL,
                description=(
                    f"电流异常: {telemetry.current}A 超过最大阈值 "
                    f"{thresholds.max_current}A"
                ),
                telemetry=telemetry,
                now=now,
            )
            detected.append(anomaly)

        elif telemetry.current > rated_current * thresholds.current_spike_ratio:
            anomaly = self._create_anomaly(
                charger_id=charger_id,
                anomaly_type=AnomalyType.OVERCURRENT,
                severity=MaintenancePriority.HIGH,
                description=(
                    f"电流突增: {telemetry.current}A 超过额定电流"
                    f"{rated_current}A的{thresholds.current_spike_ratio}倍"
                ),
                telemetry=telemetry,
                now=now,
            )
            detected.append(anomaly)

        # 过温检测
        if telemetry.temperature > thresholds.max_temperature:
            severity = (
                MaintenancePriority.CRITICAL
                if telemetry.temperature > thresholds.max_temperature + 15
                else MaintenancePriority.HIGH
            )
            anomaly = self._create_anomaly(
                charger_id=charger_id,
                anomaly_type=AnomalyType.OVERTEMPERATURE,
                severity=severity,
                description=(
                    f"温度异常: {telemetry.temperature}°C 超过阈值 "
                    f"{thresholds.max_temperature}°C"
                ),
                telemetry=telemetry,
                now=now,
            )
            detected.append(anomaly)

        # 电压异常检测
        if telemetry.voltage < thresholds.min_voltage or telemetry.voltage > thresholds.max_voltage:
            anomaly = self._create_anomaly(
                charger_id=charger_id,
                anomaly_type=AnomalyType.VOLTAGE_ABNORMAL,
                severity=MaintenancePriority.HIGH,
                description=(
                    f"电压异常: {telemetry.voltage}V，"
                    f"正常范围 [{thresholds.min_voltage}V, {thresholds.max_voltage}V]"
                ),
                telemetry=telemetry,
                now=now,
            )
            detected.append(anomaly)

        # 通信故障检测（心跳超时）
        if telemetry.status != ChargerStatus.OFFLINE:
            heartbeat_age = (now - telemetry.last_heartbeat).total_seconds()
            if heartbeat_age > thresholds.heartbeat_timeout_seconds:
                anomaly = self._create_anomaly(
                    charger_id=charger_id,
                    anomaly_type=AnomalyType.COMMUNICATION_FAILURE,
                    severity=MaintenancePriority.HIGH,
                    description=(
                        f"心跳超时: 最近心跳 {heartbeat_age:.0f}秒前，"
                        f"超过阈值 {thresholds.heartbeat_timeout_seconds}秒"
                    ),
                    telemetry=telemetry,
                    now=now,
                )
                detected.append(anomaly)

        # 错误码检测
        if telemetry.error_codes:
            for error_code in telemetry.error_codes:
                anomaly = self._create_anomaly(
                    charger_id=charger_id,
                    anomaly_type=AnomalyType.INSULATION_FAILURE,
                    severity=MaintenancePriority.MEDIUM,
                    description=f"充电桩上报错误码: {error_code}",
                    telemetry=telemetry,
                    now=now,
                )
                detected.append(anomaly)

        # 触发异常回调
        for anomaly in detected:
            self._anomaly_records[anomaly.anomaly_id] = anomaly
            for cb in self._anomaly_callbacks:
                try:
                    cb(anomaly)
                except Exception:
                    pass

        return detected

    def create_maintenance_ticket(
        self,
        charger_id: str,
        anomaly_ids: List[str],
        priority: MaintenancePriority,
        description: str,
        assigned_technician: Optional[str] = None,
    ) -> MaintenanceTicket:
        """创建维护工单

        根据异常记录创建维护工单，支持指定优先级和指派技术人员。

        Args:
            charger_id: 充电桩ID
            anomaly_ids: 关联的异常记录ID列表
            priority: 优先级
            description: 工单描述
            assigned_technician: 指派的技术人员

        Returns:
            MaintenanceTicket: 创建的维护工单

        Raises:
            ValueError: 参数不合法
        """
        if not anomaly_ids:
            raise ValueError("工单必须关联至少一条异常记录")

        # 校验异常记录存在
        missing = [aid for aid in anomaly_ids if aid not in self._anomaly_records]
        if missing:
            raise ValueError(f"以下异常记录不存在: {missing}")

        ticket_id = f"mt_{uuid.uuid4().hex[:12]}"
        ticket = MaintenanceTicket(
            ticket_id=ticket_id,
            charger_id=charger_id,
            anomaly_ids=anomaly_ids,
            priority=priority,
            description=description,
            assigned_technician=assigned_technician,
        )
        self._maintenance_tickets[ticket_id] = ticket

        # 标记异常为已确认
        for aid in anomaly_ids:
            self._anomaly_records[aid].acknowledged = True

        # 将充电桩状态设为维护中
        if charger_id in self._telemetry_cache:
            self._telemetry_cache[charger_id].status = ChargerStatus.MAINTENANCE

        return ticket

    def remote_reset(
        self,
        charger_id: str,
        reset_type: str = "soft_reset",
        issued_by: str = "system",
        timeout_seconds: int = 30,
    ) -> ResetCommand:
        """远程重置充电桩

        支持软重置（重启应用层）、硬重置（重启硬件）、
        固件更新重置。发送重置命令后等待充电桩响应。

        Args:
            charger_id: 充电桩ID
            reset_type: 重置类型 (soft_reset/hard_reset/firmware_update)
            issued_by: 操作人
            timeout_seconds: 等待响应超时时间

        Returns:
            ResetCommand: 重置命令记录

        Raises:
            ValueError: 充电桩不在线或重置类型不合法
        """
        valid_reset_types = {"soft_reset", "hard_reset", "firmware_update"}
        if reset_type not in valid_reset_types:
            raise ValueError(
                f"无效的重置类型: {reset_type}，"
                f"可选: {valid_reset_types}"
            )

        telemetry = self._telemetry_cache.get(charger_id)
        if telemetry and telemetry.status == ChargerStatus.OFFLINE:
            raise ValueError(f"充电桩 {charger_id} 离线，无法执行远程重置")

        if telemetry and telemetry.status == ChargerStatus.CHARGING:
            raise ValueError(
                f"充电桩 {charger_id} 正在充电中，"
                f"请先停止充电会话再执行重置"
            )

        command_id = f"rst_{uuid.uuid4().hex[:12]}"
        command = ResetCommand(
            command_id=command_id,
            charger_id=charger_id,
            reset_type=reset_type,
            issued_by=issued_by,
            issued_at=datetime.now(),
        )
        self._reset_commands[command_id] = command

        # 模拟发送重置命令并等待响应
        result = self._execute_remote_reset(
            charger_id, reset_type, timeout_seconds
        )
        command.result = result
        command.completed_at = datetime.now()

        if result == ResetResult.SUCCESS:
            if charger_id in self._telemetry_cache:
                self._telemetry_cache[charger_id].status = ChargerStatus.ONLINE
        elif result == ResetResult.NOT_RESPONSIVE:
            if charger_id in self._telemetry_cache:
                self._telemetry_cache[charger_id].status = ChargerStatus.FAULT
            # 重置无响应，自动创建高优先级工单
            self.create_maintenance_ticket(
                charger_id=charger_id,
                anomaly_ids=[],
                priority=MaintenancePriority.CRITICAL,
                description=(
                    f"远程重置({reset_type})后充电桩无响应，"
                    f"需要现场检查"
                ),
            )

        return command

    # ---- 内部辅助方法 ----

    def _create_anomaly(
        self,
        charger_id: str,
        anomaly_type: AnomalyType,
        severity: MaintenancePriority,
        description: str,
        telemetry: TelemetryData,
        now: datetime,
    ) -> AnomalyRecord:
        """创建异常记录"""
        snapshot = {
            "voltage": telemetry.voltage,
            "current": telemetry.current,
            "temperature": telemetry.temperature,
            "power": telemetry.power,
            "status": telemetry.status.value,
            "error_codes": telemetry.error_codes,
        }
        return AnomalyRecord(
            anomaly_id=f"anm_{uuid.uuid4().hex[:12]}",
            charger_id=charger_id,
            anomaly_type=anomaly_type,
            severity=severity,
            description=description,
            telemetry_snapshot=snapshot,
            detected_at=now,
        )

    def _execute_remote_reset(
        self, charger_id: str, reset_type: str, timeout: int
    ) -> ResetResult:
        """执行远程重置（模拟）"""
        # 实际场景中通过MQTT/HTTP发送命令到充电桩控制器
        # 这里模拟重置结果
        telemetry = self._telemetry_cache.get(charger_id)
        if telemetry is None:
            return ResetResult.NOT_RESPONSIVE

        # 硬重置和固件更新有更高失败概率
        if reset_type == "hard_reset":
            # 模拟硬重置可能无响应的情况
            return ResetResult.SUCCESS
        elif reset_type == "firmware_update":
            return ResetResult.SUCCESS
        return ResetResult.SUCCESS

    def update_charger_config(
        self, charger_id: str, config: Dict
    ) -> Dict:
        """远程更新充电桩配置

        Args:
            charger_id: 充电桩ID
            config: 配置字典，可包含 max_current, charging_mode 等

        Returns:
            Dict: 更新结果
        """
        telemetry = self._telemetry_cache.get(charger_id)
        if telemetry and telemetry.status == ChargerStatus.OFFLINE:
            raise ValueError(f"充电桩 {charger_id} 离线，无法更新配置")

        # 校验配置项
        allowed_keys = {"max_current", "charging_mode", "power_limit", "schedule"}
        invalid_keys = set(config.keys()) - allowed_keys
        if invalid_keys:
            raise ValueError(f"不支持的配置项: {invalid_keys}")

        if charger_id not in self._charger_configs:
            self._charger_configs[charger_id] = {}
        self._charger_configs[charger_id].update(config)

        # 如果更新了额定电流，同步到额定电流表
        if "max_current" in config:
            self._rated_currents[charger_id] = config["max_current"]

        return {
            "charger_id": charger_id,
            "updated_config": config,
            "updated_at": datetime.now().isoformat(),
        }

    def get_charger_status(self, charger_id: str) -> Optional[Dict]:
        """获取充电桩当前状态"""
        telemetry = self._telemetry_cache.get(charger_id)
        if telemetry is None:
            return None
        return {
            "charger_id": charger_id,
            "status": telemetry.status.value,
            "voltage": telemetry.voltage,
            "current": telemetry.current,
            "temperature": telemetry.temperature,
            "power": telemetry.power,
            "firmware_version": telemetry.firmware_version,
            "last_heartbeat": telemetry.last_heartbeat.isoformat(),
            "error_codes": telemetry.error_codes,
        }

    def get_open_tickets(self, charger_id: Optional[str] = None) -> List[Dict]:
        """获取未关闭的维护工单"""
        tickets = self._maintenance_tickets.values()
        if charger_id:
            tickets = [t for t in tickets if t.charger_id == charger_id]
        open_tickets = [t for t in tickets if t.status in ("open", "in_progress")]
        return [
            {
                "ticket_id": t.ticket_id,
                "charger_id": t.charger_id,
                "priority": t.priority.value,
                "description": t.description,
                "status": t.status,
                "assigned_technician": t.assigned_technician,
                "created_at": t.created_at.isoformat(),
            }
            for t in open_tickets
        ]

    def register_anomaly_callback(self, callback: callable):
        """注册异常检测回调"""
        self._anomaly_callbacks.append(callback)

    def set_rated_current(self, charger_id: str, rated_current: float):
        """设置充电桩额定电流"""
        self._rated_currents[charger_id] = rated_current
```

## 异常场景补充

### 场景：远程诊断误报导致停机
```
trigger: 诊断系统因传感器校准偏移或网络抖动，将正常运行的充电桩误判为过温/过流异常，自动触发停机保护，导致正在充电的用户被中断充电
detection:
  1. 对比同一充电桩多个传感器读数，若温度传感器A报85°C而传感器B报45°C，差异超过阈值则标记为疑似误报
  2. 监控停机后遥测数据是否迅速回归正常范围，若5分钟内所有指标恢复正常，标记为疑似误报
  3. 统计各充电桩的误报率，单桩月误报率超过5%即触发告警
  4. 分析误报发生的时间模式，如集中出现在网络高峰期则指向通信问题
handling:
  1. 检测到疑似误报后，不立即停机，而是将异常等级从CRITICAL降级为MEDIUM并发送确认请求
  2. 等待下一次遥测数据上报（通常10-30秒），若数据回归正常则撤销停机指令
  3. 若已触发停机，优先恢复充电桩运行，同时向受影响用户推送道歉通知及补偿（如免费充电时长）
  4. 对误报的充电桩标记"诊断降级"状态，72小时内异常需人工二次确认才执行停机
prevention:
  1. 异常检测引入多数据源交叉验证：同时参考温度传感器、电流传感器和功率计算值，单一指标异常不触发停机
  2. 设置异常持续时间阈值：异常指标需连续3次遥测上报均超阈值才判定为真实异常
  3. 区分"警告"和"停机"两级响应：警告仅通知运维，不停机；仅当多指标同时异常时才自动停机
  4. 定期校准传感器，对温度/电流传感器设置漂移检测，漂移超过5%自动标记需校准
  5. 网络抖动场景使用遥测数据中值滤波，避免单次毛刺触发误判
```

### 场景：远程重置后充电桩无响应
```
trigger: 运维人员对故障充电桩执行远程硬重置或固件更新后，充电桩进入无响应状态，心跳断开，所有遥测数据停止上报
detection:
  1. remote_reset 方法设置超时计时器，若重置命令发出后30秒内未收到充电桩重启心跳，判定为无响应
  2. 监控充电桩状态变更序列：重置前ONLINE -> 重置中MAINTENANCE -> 期望ONLINE，若停留在FAULT或OFFLINE超过2分钟则告警
  3. 与同站点其他充电桩对比，排除站点级断电导致的批量离线
  4. 检查重置命令日志，确认命令是否成功到达充电桩控制器
handling:
  1. 首次无响应：等待60秒后自动发起第二次软重置尝试
  2. 第二次仍无响应：创建CRITICAL优先级维护工单，标记"需现场处理"，指派最近的技术人员
  3. 通知站点运营方将该充电桩标记为"暂停服务"，在用户App和导航中隐藏
  4. 调取重置前的最后遥测快照，分析可能导致重置失败的原因（如固件版本不兼容）
  5. 若是固件更新导致的无响应，准备回滚固件包，待技术人员到场后通过本地接口恢复
prevention:
  1. 远程重置前强制备份当前固件版本和配置，确保可回滚
  2. 固件更新采用A/B分区机制，新固件写入备用分区，启动失败自动切换回旧分区
  3. 重置操作引入"看门狗"机制：充电桩内置硬件看门狗，超时未完成重置自动恢复到上一稳定状态
  4. 远程重置前检查充电桩电池电量/UPS状态，确保有足够电力完成重启流程
  5. 限制每日单桩远程重置次数（最多3次），防止反复重置加剧故障
```

## 充电桩智能调度与负载均衡完整实现

```python
class ChargingSchedulerService:
    """智能调度：需求预测 → 负载均衡 → 谷电利用 → 优先级排队"""

    PRIORITY_LEVELS = {
        "emergency": {"max_wait_minutes": 10, "description": "紧急（电量 < 10%）"},
        "high": {"max_wait_minutes": 30, "description": "高优先级（电量 < 20%）"},
        "normal": {"max_wait_minutes": 60, "description": "普通"},
        "low": {"max_wait_minutes": 120, "description": "低优先级（预约充电）"},
    }

    def request_charging(self, vehicle_id, station_id, target_soc_pct, priority="normal"):
        """请求充电"""
        # 1. 查找可用充电桩
        available_piles = self.db.query(
            "SELECT * FROM charging_piles "
            "WHERE station_id = %s AND status = 'available' "
            "AND connector_type IN ("
            "  SELECT connector_type FROM vehicles WHERE id = %s)",
            station_id, vehicle_id)

        if not available_piles:
            # 加入排队
            return self._join_queue(vehicle_id, station_id, target_soc_pct, priority)

        # 2. 选择最优充电桩（负载最低）
        best_pile = min(available_piles,
            key=lambda p: float(self.redis.get(f"pile_load:{p['id']}") or 0))

        # 3. 计算充电时间
        vehicle = self.db.get_vehicle(vehicle_id)
        current_soc = vehicle.get("current_soc_pct", 50)
        battery_capacity_kwh = vehicle.get("battery_capacity_kwh", 60)
        energy_needed = battery_capacity_kwh * (target_soc_pct - current_soc) / 100

        charging_power_kw = min(best_pile["max_power_kw"],
                               vehicle.get("max_charging_power_kw", 60))
        estimated_minutes = energy_needed / charging_power_kw * 60

        # 4. 创建充电会话
        session_id = str(uuid4())
        self.db.insert("charging_sessions", {
            "session_id": session_id,
            "vehicle_id": vehicle_id,
            "station_id": station_id,
            "pile_id": best_pile["id"],
            "current_soc": current_soc,
            "target_soc": target_soc_pct,
            "charging_power_kw": charging_power_kw,
            "energy_needed_kwh": round(energy_needed, 2),
            "estimated_end": now() + timedelta(minutes=estimated_minutes),
            "status": "charging",
            "started_at": now()
        })

        # 5. 更新充电桩状态
        self.db.update("charging_piles",
            {"status": "charging", "current_session_id": session_id},
            {"id": best_pile["id"]})

        self.redis.set(f"pile_load:{best_pile['id']}", charging_power_kw)

        return {"session_id": session_id, "pile_id": best_pile["id"],
                "charging_power_kw": charging_power_kw,
                "estimated_minutes": round(estimated_minutes),
                "estimated_cost": round(energy_needed * self._get_current_rate(station_id), 2)}

    def optimize_station_load(self, station_id):
        """优化站点负载（削峰填谷）"""
        # 1. 获取站点变压器容量
        station = self.db.get_station(station_id)
        max_capacity_kw = station["transformer_capacity_kw"]

        # 2. 获取当前总负载
        active_sessions = self.db.query(
            "SELECT * FROM charging_sessions "
            "WHERE station_id = %s AND status = 'charging'",
            station_id)

        current_load = sum(s["charging_power_kw"] for s in active_sessions)

        # 3. 如果超载 → 降功率
        if current_load > max_capacity_kw * 0.9:
            # 按优先级排序（低优先级先降）
            sessions_sorted = sorted(active_sessions,
                key=lambda s: self._get_session_priority(s))

            for session in sessions_sorted:
                if current_load <= max_capacity_kw * 0.85:
                    break

                # 降低该会话的充电功率
                original_power = session["charging_power_kw"]
                reduced_power = original_power * 0.5

                self.db.update("charging_sessions",
                    {"charging_power_kw": reduced_power,
                     "power_reduced": True},
                    {"session_id": session["session_id"]})

                # 发送降低功率指令
                self.iot_command.send(session["pile_id"], {
                    "action": "reduce_power",
                    "target_power_kw": reduced_power
                })

                current_load -= (original_power - reduced_power)

        # 4. 如果谷电时段 → 提高低优先级充电功率
        if self._is_valley_electricity_period():
            low_priority_sessions = [s for s in active_sessions
                if self._get_session_priority(s) >= 2  # normal/low
                and s.get("power_reduced")]

            for session in low_priority_sessions:
                if current_load >= max_capacity_kw * 0.85:
                    break

                original_target = session.get("original_power_kw",
                    session["charging_power_kw"] * 2)
                self.db.update("charging_sessions",
                    {"charging_power_kw": original_target,
                     "power_reduced": False},
                    {"session_id": session["session_id"]})

                self.iot_command.send(session["pile_id"], {
                    "action": "increase_power",
                    "target_power_kw": original_target
                })

                current_load += (original_target - session["charging_power_kw"])

        return {"station_id": station_id,
                "max_capacity_kw": max_capacity_kw,
                "current_load_kw": round(current_load, 1),
                "load_pct": round(current_load / max_capacity_kw * 100, 1),
                "active_sessions": len(active_sessions)}

    def predict_station_demand(self, station_id, hours_ahead=24):
        """预测站点需求"""
        # 1. 历史同时段数据
        historical = self.db.query(
            "SELECT HOUR(started_at) as hour, "
            "COUNT(*) as session_count, "
            "AVG(energy_needed_kwh) as avg_energy "
            "FROM charging_sessions "
            "WHERE station_id = %s "
            "AND started_at > NOW() - INTERVAL 28 DAY "
            "AND DAYOFWEEK(started_at) = DAYOFWEEK(NOW()) "
            "GROUP BY HOUR(started_at) "
            "ORDER BY hour",
            station_id)

        # 2. 天气修正
        weather = self.weather_api.get_forecast(hours=hours_ahead)
        cold_weather_modifier = 1.2 if weather.get("temp_c", 20) < 5 else 1.0
        hot_weather_modifier = 1.1 if weather.get("temp_c", 20) > 35 else 1.0

        # 3. 生成预测
        predictions = []
        for h in historical:
            predicted_sessions = int(h["session_count"] * cold_weather_modifier *
                                    hot_weather_modifier)
            predicted_energy = h["avg_energy"] * predicted_sessions

            predictions.append({
                "hour": h["hour"],
                "predicted_sessions": predicted_sessions,
                "predicted_energy_kwh": round(predicted_energy, 1)
            })

        return {"station_id": station_id, "predictions": predictions,
                "weather_modifier": {
                    "cold": cold_weather_modifier,
                    "hot": hot_weather_modifier
                }}

    def _join_queue(self, vehicle_id, station_id, target_soc, priority):
        """加入排队"""
        queue_id = str(uuid4())
        position = self.redis.incr(f"queue_position:{station_id}")

        self.db.insert("charging_queue", {
            "queue_id": queue_id,
            "vehicle_id": vehicle_id,
            "station_id": station_id,
            "target_soc": target_soc,
            "priority": priority,
            "position": position,
            "status": "waiting",
            "joined_at": now()
        })

        max_wait = self.PRIORITY_LEVELS.get(priority, {}).get("max_wait_minutes", 60)

        return {"status": "queued", "queue_id": queue_id,
                "position": position,
                "estimated_wait_minutes": position * 30,
                "max_wait_minutes": max_wait}

    def _get_session_priority(self, session):
        """获取会话优先级（数字越大优先级越低）"""
        priority_map = {"emergency": 0, "high": 1, "normal": 2, "low": 3}
        return priority_map.get(session.get("priority", "normal"), 2)

    def _get_current_rate(self, station_id):
        """获取当前电价"""
        if self._is_valley_electricity_period():
            return 0.3  # 谷电
        elif self._is_peak_electricity_period():
            return 1.2  # 峰电
        else:
            return 0.7  # 平电

    def _is_valley_electricity_period(self):
        """是否谷电时段"""
        hour = now().hour
        return 23 <= hour or hour < 7

    def _is_peak_electricity_period(self):
        """是否峰电时段"""
        hour = now().hour
        return (8 <= hour <= 11) or (17 <= hour <= 21)
```

## 异常场景补充

### 场景：充电站变压器过载跳闸

```
触发：10 辆车同时快充 → 总功率超变压器容量 → 跳闸 → 所有充电中断
检测：
  1. 变压器负载率 > 95% → 即将过载
  2. 突然所有充电桩同时离线 → 跳闸
处理：
  1. 负载率 > 90% 时自动限制新充电请求
  2. 逐步降低现有会话功率
  3. 跳闸后自动恢复（依次重启充电桩）
预防：负载限制 + 功率调节 + 依次恢复
```

### 场景：充电桩故障导致排队堆积

```
触发：站点 8 个桩中 3 个故障 → 可用桩不足 → 排队 30+ → 用户等待 2 小时
检测：
  1. 故障桩数 > 总桩数 30% → 运力不足
  2. 排队长度 > 10 → 堆积
处理：
  1. 紧急派维修人员
  2. 引导用户去附近站点
  3. 低电量车辆优先充电
预防：维修 SLA + 引导分流 + 优先级调度
```

### 场景：预约充电用户不到导致桩空占

```
触发：用户预约了充电桩 → 但未按时到达 → 充电桩空置 → 其他用户无法使用
检测：
  1. 预约时间已过但桩仍空闲 → 用户爽约
  2. 充电桩被预约但无会话 → 资源浪费
处理：
  1. 预约超时 15 分钟自动取消
  2. 取消后自动通知排队用户
  3. 多次爽约 → 限制预约功能
预防：超时取消 + 排队通知 + 爽约限制
```
