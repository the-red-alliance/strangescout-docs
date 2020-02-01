disqus:
<!-- Disables disqus comment system for this page -->

## 2020 Infinite Recharge

``` yaml
---
gameInfo: # general game info
  name: "2020 FIRST Infinite Recharge" # game name
  program: FRC # game program (frc/ftc)
  year: 2020 # game year
  duration: 150 # game duration
events: # event keys to be used at
  - 2020ncpem
  - 2020ncgui
positions: # possible starting positions
  - display: Left # position display name
    key: left # position key
  - display: Middle
    key: middle
  - display: Right
    key: right
loadouts: # robot starting loadouts
  - display: "Power Cell" # loadout display name
    events: # list of events loadout triggers
      - get_cell
  - display: "2 Power Cells"
    events:
      - get_cell
      - get_cell
  - display: "3 Power Cells"
    events:
      - get_cell
      - get_cell
      - get_cell
scout: # scouting template data
  run: # list of game elements for scouting a run
    - type: multi_item # element type
      activeTime: 0 # T+ time to activate the element
      display: "Power Cells" # element display name (multi item)
      key: "power_cells" # element key (multi item)
      max: 5 # max to be held (multi item)
      get: # get event (multi item)
        display: "Get Power Cell" # get event display name (multi item)
        key: get_cell # get event key (multi item)
      endDisable: true # disable at end of match
      children: # event children (destinations)
        - display: "Upper Power Ports"
          key: upper_cell
        - display: "Lower Power Port"
          key: lower_cell
        - display: "Dropped Cell"
          key: drop_cell
    - type: single_item # single item
      activeTime: 0
      canHold: false # can't hold
      ignoreHold: true # doable when holding
      endDisable: true # disables at the end
      display: "Start Control Panel"
      key: start_panel
      children:
        - display: "Successful Spin"
          key: successful_panel
        - display: "Failed Spin"
          key: failed_panel
    - type: single_item # single item
      activeTime: 120 # activates at T+120 / T-30
      canHold: false # can't hold
      ignoreHold: true # doable while holding
      endDisable: false # doesn't disable at the end
      singleUse: true # can only occur once
      display: "Start Hang"
      key: start_hang
      children:
        - display: "Successful Hang"
          key: successful_hang
        - display: "Failed Hang"
          key: failed_hang
    - type: duration # duration element
      activeTime: 0
      key: defense
      startDisplay: "Start Defending"
      startKey: start_defend
      endDisplay: "Stop Defending"
      endKey: end_defend
      ignoreHold: true
      endDisable: true
  pit: # pit scouting
    - type: number # a number input
      name: Ground Clearance (inches) # the display name
      key: ground_clearance # the field key
      required: true # is it required
    - type: boolean
      name: Control Panel Clearance
      key: control_panel
    - type: select # a select input
      name: Drivetrain # the display name
      key: drivetrain # the input key
      options: # the select options
        - name: Kit Bot # option display
          key: kit_bot # option value
        - name: Swerve
          key: swerve
        - name: West Coast
          key: west_coast
        - name: Mecanum
          key: mecanum
        - name: All Omni
          key: all_omni
        - name: Pneumatic
          key: pneumatic
        - name: Eight Wheel
          key: eight_wheel
        - name: Treads
          key: treads
        - name: Other (in notes)
          key: other
      required: true
```