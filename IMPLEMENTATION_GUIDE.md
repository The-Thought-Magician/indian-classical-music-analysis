# Implementation Guide: Improving Indian Classical Music Generation

## Overview

This guide provides concrete implementation steps and code examples to address the issues identified in ANALYSIS.md. It's organized from highest ROI (quick wins) to highest effort (long-term solutions).

---

## Phase 1: AI Prompt Improvements (40% Impact)

### Current System Prompt (Problematic)

```
"Generate a musical composition..."
- Emphasizes minimum note counts
- Uses "classical" = "slow and sparse"
- No mention of ornamentation
- No cultural context
```

### Improved System Prompt

```markdown
# Indian Classical Music Composition System Prompt

You are an expert in Indian classical music composition. Generate structured MIDI-compatible 
compositions that authentically represent Indian ragas and talas.

## Required Output Format

Return a JSON object with this structure:

```json
{
  "metadata": {
    "title": "composition_title",
    "duration_minutes": 8.5,
    "tempo_bpm": 80,
    "primary_raga": "Raga Name",
    "tala": "Tintal (16 beats)"
  },
  "instruments": [
    {
      "name": "Sitar",
      "midi_program": 104,
      "midi_channel": 0,
      "notes": [...]
    }
  ]
}
```

## Key Constraints for Indian Classical Music

### 1. Note Density (CRITICAL)
- **Minimum:** 300-400 total notes per melody instrument per 8-minute piece
- **Target:** 500-800 notes
- **For Sitar/Bansuri:** 60-100 notes per minute
- **Reasoning:** Indian classical features rapid melodic passages, not Western sparse compositions

### 2. Note Duration Distribution (CRITICAL)
- **Very Short (0.125-0.25 beats):** 20% of notes
- **Short (0.25-0.5 beats):** 30% of notes
- **Medium (0.5-1.0 beats):** 30% of notes
- **Long (1.0-2.0 beats):** 15% of notes
- **Very Long (2.0+ beats):** 5% of notes
- **Never:** No note longer than 4 beats (except tanpura)
- **Reasoning:** Prevents the "dragging" effect; enables rhythmic vitality

### 3. Ornamentation (DEFINES INDIAN CLASSICAL)
Include ornamentation in 40-60% of notes:

#### Pitch Bends (Meend)
- Use for 30-40% of notes
- Direction: upward (ascending meend), downward (descending meend), or oscillating
- Timing: smooth over 0.3-0.8 beats
- Example: Note C4, slide up to D4 over 0.5 beats

```json
{
  "note_name": "C4",
  "start_time": 2.0,
  "duration": 0.75,
  "velocity": 75,
  "pitch_bend": {
    "start_semitones": 0,
    "end_semitones": 2,
    "curve": "linear",
    "duration_beats": 0.5
  }
}
```

#### Control Change (CC) Events (Vibrato, Tremolo)
- CC1 (Modulation) for vibrato depth: ramped from 0 to 80 over note duration
- CC11 (Expression) for dynamic variation
- Apply to 20-30% of long notes (1.0+ beats)

```json
{
  "note_name": "G4",
  "start_time": 3.5,
  "duration": 2.0,
  "velocity": 70,
  "cc_events": [
    {
      "controller": 1,
      "start_time": 3.5,
      "end_time": 5.5,
      "start_value": 0,
      "end_value": 80,
      "curve": "exponential"
    }
  ]
}
```

#### Grace Notes (Krintan)
- Add 1-4 rapid notes before a main note
- Duration: 0.1-0.2 beats each
- Use adjacent scale degrees

```json
{
  "note_name": "D4",
  "start_time": 5.0,
  "grace_notes": [
    {"note_name": "C4", "duration": 0.1},
    {"note_name": "C#4", "duration": 0.1}
  ]
}
```

### 4. Raga-Specific Constraints
When generating for a specific raga, adhere to:
- **Aroha (ascending):** Specify 5-7 note sequence
- **Avaroha (descending):** Different sequence (usually)
- **Pakad (catch phrase):** Inject every 4-8 bars
- **Vadi emphasis:** Emphasize vadi note with:
  - Longer duration (1.5-2.0 beats)
  - Appearance frequency (30-40% of notes)
  - Ornamentation around it
- **Samvadi:** Secondary importance (15-20% frequency)
- **Avoid notes:** Never use notes outside aroha/avaroha

**Example: Raga Bhairav**
```json
{
  "aroha": ["Sa", "Re", "Ga", "Ma", "Pa", "Dha", "Ni", "Sa"],
  "avaroha": ["Sa", "Ni", "Dha", "Pa", "Ma", "Ga", "Re", "Sa"],
  "vadi": "Ma",
  "samvadi": "Sa",
  "pakad_phrases": [
    ["Ga", "Ma", "Pa"],
    ["Re", "Ga", "Ma"],
    ["Dha", "Ni", "Sa"]
  ],
  "best_time": "Early morning",
  "season": "Spring"
}
```

### 5. Tanpura Implementation (Drone Foundation)

```json
{
  "name": "Tanpura",
  "midi_channel": 9,
  "notes": [
    {
      "note_name": "C3",
      "start_time": 0,
      "duration": 520,
      "velocity": 50,
      "midi_note": 36
    },
    {
      "note_name": "G3",
      "start_time": 0,
      "duration": 520,
      "velocity": 50,
      "midi_note": 43
    },
    {
      "note_name": "C4",
      "start_time": 0.05,
      "duration": 520,
      "velocity": 45,
      "midi_note": 48
    },
    {
      "note_name": "G3",
      "start_time": 0.1,
      "duration": 520,
      "velocity": 48,
      "midi_note": 43,
      "note": "Slightly detuned (±5 cents) for beating effect"
    }
  ]
}
```

### 6. Tabla with Tala Patterns (Rhythmic Cycle)

#### Tintal (16-beat cycle)
```json
{
  "name": "Tabla",
  "midi_channel": 9,
  "tala": "Tintal",
  "tala_cycle_beats": 16,
  "bols": ["dha", "dha", "dhin", "dhin", "dha", "dha", "dhin", "dhin", 
           "dha", "dha", "dhin", "dhin", "dha", "dha", "dhin", "dhin"],
  "notes": [
    {
      "start_time": 0.0,
      "duration": 0.25,
      "note_name": "D3",
      "midi_note": 38,
      "velocity": 80,
      "stroke": "dha (right hand open blow)"
    },
    {
      "start_time": 0.25,
      "duration": 0.25,
      "note_name": "D3",
      "midi_note": 38,
      "velocity": 75,
      "stroke": "dha (right hand open blow)"
    },
    {
      "start_time": 0.5,
      "duration": 0.25,
      "note_name": "E3",
      "midi_note": 40,
      "velocity": 85,
      "stroke": "dhin (struck)"
    },
    {
      "start_time": 0.75,
      "duration": 0.25,
      "note_name": "E3",
      "midi_note": 40,
      "velocity": 80,
      "stroke": "dhin (struck)"
    }
  ]
}
```

**Mapping tabla strokes to MIDI notes:**
- **D3 (38):** Bayan bass (open)
- **E3 (40):** Dayan treble (closed/center)
- **F3 (41):** Dayan treble (open/edge)
- **C3 (36):** Bass muted
- **G3 (43):** Combined (both drums light touch)

### 7. Melodic Development (Vistaar)
Ensure melodic progression:
- **First 1/3:** Introduce notes in lower register (mandra saptak)
- **Middle 1/3:** Move to middle register (madhya saptak), introduce vadi-samvadi
- **Last 1/3:** Explore upper register (tara saptak) with ornamentation, return to lower
- **Phrase structure:** 2-4 beat phrases with 0.5-1 beat breath space
- **Repetition with variation:** Same phrase appears 2-3 times with slight melodic changes

### 8. Structure (Alap-Jor-Gat approximation)

**For 8-10 minute composition:**
- **Alap (0-2 min):** Slow, pulse-free, sparse notes, long durations
- **Transition (2-3 min):** Introduce pulse, gradual note density increase
- **Jor (3-4 min):** Medium tempo, establish rhythm, 60-80 notes/min
- **Gat (4-8 min):** Fast, tabla enters with tala patterns, 80-120 notes/min, ornamentation heavy
- **Conclusion (8+ min):** Return to slower pace, resolve on Sa

---

## Phase 2: Data Schema Fixes (Immediate)

### Fix: Accept Both "pitch" and "note_name"

```python
# Parser update
def parse_note(note_dict):
    note_name = note_dict.get('note_name') or note_dict.get('pitch')
    return note_name
```

### New Schema with Expression Support

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "note_name": {
      "type": "string",
      "pattern": "^[A-G][#b]?[0-9]$"
    },
    "start_time": {"type": "number"},
    "duration": {"type": "number"},
    "velocity": {"type": "integer", "minimum": 0, "maximum": 127},
    "pitch_bend": {
      "type": "object",
      "properties": {
        "start_semitones": {"type": "number"},
        "end_semitones": {"type": "number"},
        "duration_beats": {"type": "number"},
        "curve": {"type": "string", "enum": ["linear", "exponential", "logarithmic"]}
      }
    },
    "cc_events": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "controller": {"type": "integer", "minimum": 0, "maximum": 119},
          "start_time": {"type": "number"},
          "end_time": {"type": "number"},
          "start_value": {"type": "integer", "minimum": 0, "maximum": 127},
          "end_value": {"type": "integer", "minimum": 0, "maximum": 127},
          "curve": {"type": "string"}
        }
      }
    },
    "grace_notes": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "note_name": {"type": "string"},
          "duration": {"type": "number"}
        }
      }
    }
  },
  "required": ["note_name", "start_time", "duration", "velocity"]
}
```

---

## Phase 3: Post-Processing Ornamentation (Quick Win)

If AI doesn't generate ornamentation, add it programmatically:

```python
def add_ornamentation(notes, ornamentation_probability=0.4):
    """
    Add meend (pitch bend) and vibrato to notes randomly.
    """
    import random
    
    for note in notes:
        if random.random() < ornamentation_probability:
            note_duration = note['duration']
            
            # Add pitch bend for longer notes
            if note_duration >= 0.5:
                direction = random.choice(['up', 'down', 'both'])
                bend_semitones = random.randint(1, 3)
                
                if direction == 'up':
                    note['pitch_bend'] = {
                        'start_semitones': 0,
                        'end_semitones': bend_semitones,
                        'duration_beats': note_duration * 0.6,
                        'curve': 'exponential'
                    }
                elif direction == 'down':
                    note['pitch_bend'] = {
                        'start_semitones': 0,
                        'end_semitones': -bend_semitones,
                        'duration_beats': note_duration * 0.6,
                        'curve': 'exponential'
                    }
                else:  # Both directions (gamaka effect)
                    note['pitch_bend'] = {
                        'start_semitones': -bend_semitones,
                        'end_semitones': bend_semitones,
                        'duration_beats': note_duration * 0.8,
                        'curve': 'sine'  # Oscillating
                    }
            
            # Add vibrato (CC1) for very long notes
            if note_duration >= 1.0:
                note['cc_events'] = [
                    {
                        'controller': 1,
                        'start_time': note['start_time'] + 0.2,
                        'end_time': note['start_time'] + note_duration - 0.1,
                        'start_value': 0,
                        'end_value': 80,
                        'curve': 'linear'
                    }
                ]
    
    return notes
```

---

## Phase 4: Raga Template Database

```python
RAGA_DATABASE = {
    "Bhairav": {
        "aroha": ["Sa", "Re", "Ga", "Ma", "Pa", "Dha", "Ni", "Sa"],
        "avaroha": ["Sa", "Ni", "Dha", "Pa", "Ma", "Ga", "Re", "Sa"],
        "vadi": "Ma",
        "samvadi": "Sa",
        "pakad": [["Ga", "Ma", "Pa"], ["Re", "Ga", "Ma"]],
        "time_of_day": "Early morning",
        "mood": "Heroic, martial",
        "scale_notes": {
            "Sa": 0,
            "Re": 1,
            "Ga": 3,
            "Ma": 5,
            "Pa": 7,
            "Dha": 8,
            "Ni": 11
        }
    },
    "Yaman": {
        "aroha": ["Sa", "Re", "Ga", "Ma", "Pa", "Dha", "Ni", "Sa"],
        "avaroha": ["Sa", "Ni", "Dha", "Pa", "Ma", "Ga", "Re", "Sa"],
        "vadi": "Ga",
        "samvadi": "Ni",
        "pakad": [["Ni", "Re", "Ga"], ["Re", "Sa"]],
        "time_of_day": "Evening",
        "mood": "Devotional, serene",
        "scale_notes": {
            "Sa": 0,
            "Re": 2,
            "Ga": 4,
            "Ma": 6,
            "Pa": 7,
            "Dha": 9,
            "Ni": 11
        }
    }
    # Add more ragas...
}

def get_raga_constraints(raga_name):
    return RAGA_DATABASE.get(raga_name)
```

---

## Phase 5: Tala Pattern Generator

```python
TALA_PATTERNS = {
    "Tintal": {
        "cycle_length": 16,
        "divisions": [4, 4, 4, 4],
        "bols": [
            "dha", "dha", "dhin", "dhin",
            "dha", "dha", "dhin", "dhin",
            "dha", "dha", "dhin", "dhin",
            "dha", "dha", "dhin", "dhin"
        ],
        "sam_position": 0,
        "khali_positions": [8, 12]
    },
    "Jhaptal": {
        "cycle_length": 10,
        "divisions": [2, 3, 2, 3],
        "bols": [
            "dhi", "mi",
            "ta", "ki", "ta",
            "jha", "nu",
            "dhi", "mi", "ta"
        ],
        "sam_position": 0,
        "khali_positions": [5, 7]
    },
    "Ektal": {
        "cycle_length": 12,
        "divisions": [2, 2, 2, 2, 2, 2],
        "bols": [
            "dhi", "mi",
            "ta", "ki",
            "ta", "jha",
            "nu", "dhi",
            "mi", "ta",
            "ki", "ta"
        ],
        "sam_position": 0,
        "khali_positions": [6]
    }
}

def generate_tabla_pattern(tala_name, duration_beats, tempo_bpm):
    """
    Generate tabla rhythm pattern for specified tala and duration.
    """
    tala = TALA_PATTERNS[tala_name]
    cycle_length = tala['cycle_length']
    beats_per_note = 1.0  # Quarter beat
    
    notes = []
    current_beat = 0
    
    while current_beat < duration_beats:
        cycle_position = int(current_beat) % cycle_length
        bol = tala['bols'][cycle_position]
        
        # Map bols to MIDI notes
        midi_note = bol_to_midi(bol)
        velocity = 85 if cycle_position == tala['sam_position'] else 70
        
        notes.append({
            "start_time": current_beat,
            "duration": beats_per_note,
            "note_name": midi_note,
            "velocity": velocity,
            "bol": bol
        })
        
        current_beat += beats_per_note
    
    return notes

def bol_to_midi(bol):
    """Map tabla bol to MIDI note."""
    BOL_MAP = {
        "dha": "D3",
        "dhin": "E3",
        "mi": "E3",
        "ta": "E3",
        "ki": "F3",
        "jha": "D3",
        "nu": "C3"
    }
    return BOL_MAP.get(bol, "D3")
```

---

## Phase 6: Testing and Validation

### Validation Checklist

```python
def validate_composition(composition):
    """
    Validate composition against Indian classical music criteria.
    """
    results = {
        "valid": True,
        "issues": [],
        "metrics": {}
    }
    
    # Check note density
    total_notes = sum(len(inst['notes']) for inst in composition['instruments'])
    duration_minutes = composition['metadata']['duration_minutes']
    notes_per_minute = total_notes / duration_minutes
    results['metrics']['notes_per_minute'] = notes_per_minute
    
    if notes_per_minute < 60:
        results['issues'].append(f"Low note density: {notes_per_minute:.1f} notes/min (target: 80+)")
        results['valid'] = False
    
    # Check note durations
    durations = []
    for inst in composition['instruments']:
        for note in inst['notes']:
            durations.append(note['duration'])
    
    avg_duration = sum(durations) / len(durations)
    results['metrics']['avg_note_duration'] = avg_duration
    
    if avg_duration > 1.5:
        results['issues'].append(f"Notes too long: avg {avg_duration:.2f} beats (target: <1.0)")
        results['valid'] = False
    
    # Check ornamentation
    ornamented_notes = 0
    for inst in composition['instruments']:
        for note in inst['notes']:
            if 'pitch_bend' in note or 'cc_events' in note:
                ornamented_notes += 1
    
    ornamentation_ratio = ornamented_notes / total_notes if total_notes > 0 else 0
    results['metrics']['ornamentation_ratio'] = ornamentation_ratio
    
    if ornamentation_ratio < 0.2:
        results['issues'].append(f"Low ornamentation: {ornamentation_ratio*100:.1f}% (target: 40%+)")
        results['valid'] = False
    
    # Check tanpura
    tanpura = next((inst for inst in composition['instruments'] if inst['name'] == 'Tanpura'), None)
    if not tanpura:
        results['issues'].append("Missing tanpura drone")
        results['valid'] = False
    elif len(tanpura['notes']) < 4:
        results['issues'].append(f"Tanpura too sparse: {len(tanpura['notes'])} notes (target: 4+)")
        results['valid'] = False
    
    return results
```

---

## Summary: Quick Implementation Checklist

- [ ] **Week 1:** Update AI system prompt with all constraints from Phase 1
- [ ] **Week 1:** Fix schema to accept both "pitch" and "note_name"
- [ ] **Week 2:** Implement ornamentation post-processor (Phase 3)
- [ ] **Week 2:** Create raga template database (Phase 4)
- [ ] **Week 3:** Implement tala pattern generator (Phase 5)
- [ ] **Week 3:** Create validation checklist (Phase 6)
- [ ] **Week 4:** Test with multiple prompts, iterate on system prompt
- [ ] **Long-term:** Develop SFZ instruments (Phase 7 from ANALYSIS.md)

---

*Implementation Guide v1.0*
*Last updated: 2025-01-18*
