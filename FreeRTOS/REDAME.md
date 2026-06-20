# FreeRTOS Exploration

This folder contains the material produced during my study of Real-Time Operating Systems (RTOS), with a focus on the internal architecture of FreeRTOS.  
The goal of this exploration is to understand how a small real-time kernel is structured, how its components interact, and how deterministic behavior is achieved on embedded systems.

---

## 📂 What this folder contains
- **FreeRTOS-Changes.odt** — Notes about kernel changes, internal mechanisms, and behavior observed during the study.
- **FreeRTOS-Roadmap.odt** — A roadmap of the topics explored and the next areas I plan to investigate.
- **Exploration Notes (this file)** — A high-level summary of the concepts learned.

---

## 🧠 Key Concepts Studied
- Task creation and management  
- Context switching and scheduling  
- Tick interrupt and time slicing  
- Queues, lists, and synchronization primitives  
- Memory allocation strategies (heap_1, heap_2, heap_4)  
- Timer service task  
- Deterministic behavior and real-time constraints  

---

## 🔍 Technical Insights
During this exploration I focused on:
- Understanding how the scheduler selects the next task  
- Analyzing how FreeRTOS implements ready lists and delayed lists  
- Inspecting the role of the SysTick interrupt  
- Studying how queues are implemented internally  
- Observing how FreeRTOS ensures predictable execution in constrained environments  

---

## 🚀 Next Steps
- Explore interrupt nesting and priority handling  
- Analyze FreeRTOS+Trace and runtime statistics  
- Study memory fragmentation and allocation strategies  
- Compare FreeRTOS behavior on different MCUs  

---

If you want to follow the evolution of this study, check the pull requests and commits associated with this folder.
