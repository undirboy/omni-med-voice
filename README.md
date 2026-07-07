import abc
import json
import logging
import math
import os
import time
import threading
from enum import Enum, auto
from dataclasses import dataclass, field
from typing import Dict, Any, List, Optional, Callable
import pyttsx3  # Native Offline Text-to-Speech Engine

# --- CONFIGURING SECURE ROTATING TELEMETRY LOGS ---
log_formatter = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
log_handler = logging.handlers.RotatingFileHandler("omnimed_telemetry.log", maxBytes=5 * 1024 * 1024, backupCount=3)
log_handler.setFormatter(log_formatter)
logger = logging.getLogger("OmniMedCore")
logger.setLevel(logging.INFO)
logger.addHandler(log_handler)

class Priority(Enum):
    LOW = 10
    MEDIUM = 60
    HIGH = 90
    CRITICAL = 100

@dataclass(frozen=True)
class TelemetryEvent:
    event_id: int
    timestamp: float
    source: str
    value: bool
    confidence: float
    priority: Priority
    latency_ms: float
    quality: str
    metadata: Dict[str, Any] = field(default_factory=dict)

# ==========================================================
# 1. OUTBOUND ASYNCHRONOUS ACTUATION MANAGERS
# ==========================================================
class EmergencyActuatorManager:
    """Handles real peripheral interfaces without blocking core processing layers."""
    
    @staticmethod
    def run_voice_broadcast(text_message: str):
        """Spins up a localized, completely offline TTS engine loop inside a worker thread."""
        def tts_worker():
            try:
                # Initialization must happen inside the executing worker thread scope
                engine = pyttsx3.init()
                
                # Calibration properties
                engine.setProperty('rate', 155)     # Moderate, clear pacing for clinical distress
                engine.setProperty('volume', 1.0)   # Maximized audio notification range
                
                logger.info(f"[TTS START] Emitting local audio payload: '{text_message}'")
                engine.say(text_message)
                engine.runAndWait()
                engine.stop()
            except Exception as tts_err:
                logger.error(f"TTS Actuator Core failure: {str(tts_err)}")

        # Thread decoupling prevents pyttsx3's blocking engine loop from freezing CV frames
        threading.Thread(target=tts_worker, daemon=True).start()

    @staticmethod
    def activate_siren():
        print("🚨 [SIREN ACTUATOR] Initializing localized high-frequency emergency buzzer sequence...")
        
    @staticmethod
    def unlock_and_present_medication():
        logger.warning("[SAFETY INTERLOCK] Presenting medication bay door interfaces for verification check.")
        print("🤖 [CLINICAL SAFETY GATE] Presenting medication vault drawer tray interface.")


# ==========================================================
# 2. COMMAND PATTERN ROUTING INTERFACES
# ==========================================================
class ICommand(abc.ABC):
    @abc.abstractmethod
    def execute(self) -> None: ...

class SpeakVocalPromptCommand(ICommand):
    """Encapsulates a TTS request cleanly as an auditable command object."""
    def __init__(self, announcement_text: str):
        self.announcement_text = announcement_text

    def execute(self):
        print(f"🗣️  [COMMAND DISPATCH] Voice Prompt: '{self.announcement_text}'")
        EmergencyActuatorManager.run_voice_broadcast(self.announcement_text)

class DispatchSirenCommand(ICommand):
    def execute(self): 
        EmergencyActuatorManager.activate_siren()

class PresentMedicationCommand(ICommand):
    def execute(self): 
        EmergencyActuatorManager.unlock_and_present_medication()

class CommandDispatcher:
    def __init__(self):
        self.command_history: List[ICommand] = []

    def dispatch(self, command: ICommand):
        self.command_history.append(command)
        command.execute()


# ==========================================================
# 3. EXPLICIT FINITE STATE MACHINE (FSM) WITH ENTRY/EXIT HOOKS
# ==========================================================
class FSMContext:
    def __init__(self, dispatcher: CommandDispatcher):
        self.dispatcher = dispatcher
        self.state_entered_time = time.time()

class FSMState(abc.ABC):
    def __init__(self, name: str):
        self.name = name
    def on_entry(self, ctx: FSMContext): pass
    def on_exit(self, ctx: FSMContext): pass
    @abc.abstractmethod
    def run_logic(self, ctx: FSMContext, facts: Dict[str, TelemetryEvent]) -> str: ...

class NormalState(FSMState):
    def __init__(self): super().__init__("NORMAL")
    def run_logic(self, ctx: FSMContext, facts: Dict[str, TelemetryEvent]) -> str:
        fall = facts.get("fall_detected")
        chest = facts.get("chest_hold")
        if (fall and fall.value) or (chest and chest.value):
            return "VERIFYING"
        return "NORMAL"

class VerifyingState(FSMState):
    def __init__(self): super().__init__("VERIFYING")
    
    def on_entry(self, ctx: FSMContext):
        # Trigger explicit, automated verbal diagnostic checks natively on entering state bounds
        ctx.dispatcher.dispatch(SpeakVocalPromptCommand(
            "Emergency profile suspected. Please speak clearly or hold your hand to the camera if you need help."
        ))
        
    def run_logic(self, ctx: FSMContext, facts: Dict[str, TelemetryEvent]) -> str:
        shout = facts.get("shout_detected")
        if shout and shout.value:
            return "ALERTING"
        if time.time() - ctx.state_entered_time > 8.0: # Time-bounded verification timeout
            return "ALERTING"
        return "VERIFYING"

class AlertingState(FSMState):
    def __init__(self): super().__init__("ALERTING")
    
    def on_entry(self, ctx: FSMContext):
        # Coordinate overlapping sensory actuators safely via the Command Pattern broker
        ctx.dispatcher.dispatch(SpeakVocalPromptCommand(
            "Critical distress confirmed. Activating alarms and dispatching immediate medical assistance."
        ))
        ctx.dispatcher.dispatch(DispatchSirenCommand())
        ctx.dispatcher.dispatch(PresentMedicationCommand())
        
    def run_logic(self, ctx: FSMContext, facts: Dict[str, TelemetryEvent]) -> str:
        return "WAITING_FOR_ACK"

class WaitingForAckState(FSMState):
    def __init__(self): super().__init__("WAITING_FOR_ACK")
    def run_logic(self, ctx: FSMContext, facts: Dict[str, TelemetryEvent]) -> str:
        return "WAITING_FOR_ACK"


# ==========================================================
# 4. DETERMINISTIC EXECUTION SIMULATION HARNESS
# ==========================================================
class OmniMedEngineCore:
    def __init__(self, dispatcher: CommandDispatcher):
        self.latest_state_registry: Dict[str, TelemetryEvent] = {}
        self.dispatcher = dispatcher
        self.states_map: Dict[str, FSMState] = {
            "NORMAL": NormalState(),
            "VERIFYING": VerifyingState(),
            "ALERTING": AlertingState(),
            "WAITING_FOR_ACK": WaitingForAckState()
        }
        self.current_state = self.states_map["NORMAL"]
        self.fsm_context = FSMContext(dispatcher)

    def update_fact(self, event: TelemetryEvent):
        self.latest_state_registry[event.source] = event

    def process_system_tick(self):
        next_state_name = self.current_state.run_logic(self.fsm_context, self.latest_state_registry)
        if next_state_name != self.current_state.name:
            self.current_state.on_exit(self.fsm_context)
            self.current_state = self.states_map[next_state_name]
            self.fsm_context = FSMContext(self.dispatcher)
            self.current_state.on_entry(self.fsm_context)


if __name__ == "__main__":
    print("🚀 [BOOT] Launching Architecture Harness with Integrated Offline TTS System...")
    action_dispatcher = CommandDispatcher()
    engine_core = OmniMedEngineCore(action_dispatcher)
    
    # --- SIMULATING AN ESCALATION INCIDENT TIMELINE ---
    print(f"\n[FSM TICK 1] Current System State: {engine_core.current_state.name}")
    
    # Injecting Fall Observation
    engine_core.update_fact(TelemetryEvent(
        event_id=9001, timestamp=time.time(), source="fall_detected",
        value=True, confidence=0.96, priority=Priority.HIGH, latency_ms=15.0, quality="GOOD"
    ))
    
    # This tick shifts the state from NORMAL -> VERIFYING, automatically executing the verbal query
    engine_core.process_system_tick()
    time.sleep(4.0) # Wait brief moment to simulate patient immobility
    
    print(f"\n[FSM TICK 2] Current System State: {engine_core.current_state.name}")
    
    # Injecting Vocal Shout Distress Packet
    engine_core.update_fact(TelemetryEvent(
        event_id=9002, timestamp=time.time(), source="shout_detected",
        value=True, confidence=0.92, priority=Priority.CRITICAL, latency_ms=30.0, quality="GOOD"
    ))
    
    # This tick escalates VERIFYING -> ALERTING, executing hardware siren, alerts, and verbal dispatches
    engine_core.process_system_tick()
    
    # Allow background thread worker processing buffer timeline to wrap up speech audio loops cleanly
    time.sleep(5.0)
