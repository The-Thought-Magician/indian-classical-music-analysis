# Quick Start Guide - Implement Changes in 1 Hour

## TL;DR

Your AI-generated Indian classical music sounds boring because:
1. **Only 315 notes in 8 minutes** (should be 800+) - fixes in system prompt
2. **Zero ornamentation** (0% meend/gamaka) - fixes with 1 function
3. **Wrong instruments** (tabla as Taiko drum) - fixes in config
4. **No raga structure** (using Western scales) - fixes in prompt

**Impact:** 40% of the problem comes from AI prompt. Fix it first. ROI is 10x effort.

---

## 30-Minute Implementation Plan

### Step 1: Update System Prompt (15 minutes)

**Copy the system prompt from [SYSTEM_PROMPT_TEMPLATE.md](./SYSTEM_PROMPT_TEMPLATE.md)**

Replace your current system prompt with the template provided. Key changes:
- Note density target: 300-600 notes per melody instrument
- Ornamentation requirement: 40-60% of notes must have pitch_bend or CC events
- Raga structure: Use authentic ragas, not Western scales
- Tala patterns: Tabla enters at minute 3 with proper rhythmic cycles

**Validation:** After updating, test with this prompt:
```
Generate a classical Indian music composition representing victory and celebration. 
Use Raga Bhairav and Tintal. Duration: 8 minutes.
```

Check output for:
- [ ] Note count: >300 total
- [ ] Notes per minute: 80+ for melody
- [ ] Contains pitch_bend in notes: Yes
- [ ] Contains CC events: Yes
- [ ] Uses only Bhairav notes: Yes

### Step 2: Add Ornamentation Post-Processor (10 minutes)

If AI output still lacks ornamentation, add this Python function:

```python
def add_meend_to_notes(notes, meend_probability=0.4):
    """
    Add pitch bend (meend) to notes automatically.
    """
    import random
    
    for note in notes:
        if random.random() < meend_probability and note.get('duration', 0) >= 0.5:
            # Add upward or downward meend
            direction = random.choice([-1, 1])
            bend_amount = random.randint(1, 3)  # semitones
            
            note['pitch_bend'] = {
                'start_semitones': 0,
                'end_semitones': direction * bend_amount,
                'duration_beats': min(note['duration'] * 0.6, 1.0),
                'curve': 'exponential'
            }
    
    return notes

# Usage in your pipeline:
composition['instruments'][0]['notes'] = add_meend_to_notes(
    composition['instruments'][0]['notes'],
    meend_probability=0.5
)
```

**Test:** Run on output from Step 1, verify 40%+ of notes have pitch_bend.

### Step 3: Fix Instrument Configuration (5 minutes)

Update your instrument-to-MIDI mapping:

```python
INSTRUMENT_CONFIG = {
    "Sitar": {
        "midi_program": 104,
        "midi_channel": 0,
        "description": "Sitar - main melody"
    },
    "Bansuri": {
        "midi_program": 73,
        "midi_channel": 1,
        "description": "Flute - secondary melody"
    },
    "Tabla": {
        "midi_program": 0,  # Use percussion sounds
        "midi_channel": 9,  # MIDI channel 10 (percussion)
        "note_mapping": {
            "D3": 38,  # Bayan (bass)
            "E3": 40,  # Dayan (treble)
            "F3": 41,  # Dayan (open)
            "C3": 36   # Bass muted
        }
    },
    "Tanpura": {
        "midi_program": 48,
        "midi_channel": 0,
        "description": "Drone foundation"
    }
}
```

---

## 1-Hour Extended Plan (Optional)

If you have 1 hour, add these for better results:

### Add Raga Template Database (20 minutes)

```python
RAGA_DB = {
    "Bhairav": {
        "notes": [0, 1, 3, 5, 7, 8, 11],  # semitones from root
        "vadi": 5,  # Madhyam
        "samvadi": 0,  # Shadaj
        "aroha": "Sa Re Ga Ma Pa Dha Ni Sa",
        "avaroha": "Sa Ni Dha Pa Ma Ga Re Sa",
        "pakad": [[3, 5, 7], [1, 3, 5]],  # Ga-Ma-Pa, Re-Ga-Ma
        "time": "Early morning",
        "mood": "Heroic, martial"
    },
    "Yaman": {
        "notes": [0, 2, 4, 6, 7, 9, 11],
        "vadi": 4,  # Gandhar
        "samvadi": 11,  # Nishad
        "aroha": "Sa Re Ga Ma Pa Dha Ni Sa",
        "avaroha": "Sa Ni Dha Pa Ma Ga Re Sa",
        "pakad": [[11, 2, 4], [2, 0]],  # Ni-Re-Ga, Re-Sa
        "time": "Evening",
        "mood": "Devotional, serene"
    }
}

def validate_raga_notes(notes_list, raga_name):
    """Ensure composition only uses notes from raga."""
    raga = RAGA_DB[raga_name]
    allowed_semitones = set(raga['notes'])
    
    for note in notes_list:
        # Extract semitone value from note name
        semitone = note_to_semitone(note['note_name'])
        if semitone % 12 not in allowed_semitones:
            print(f"Warning: {note['note_name']} not in {raga_name}")
```

### Add Tala Pattern Generator (20 minutes)

```python
def generate_tintal_pattern(start_time, duration, tempo_bpm, velocity_base=75):
    """
    Generate Tintal (16-beat) rhythm pattern.
    """
    bols = [
        ("dha", 38),  # MIDI note for Bayan
        ("dha", 38),
        ("dhin", 40),  # MIDI note for Dayan
        ("dhin", 40),
    ] * 4  # Repeat 4 times for 16-beat cycle
    
    notes = []
    current_time = start_time
    beat_duration = 0.25  # Quarter beat
    
    while current_time < start_time + duration:
        bol_index = int((current_time - start_time) * 4) % len(bols)
        bol, midi_note = bols[bol_index]
        
        # Accent on Sam (beat 0 of cycle)
        velocity = velocity_base + 10 if bol_index % 4 == 0 else velocity_base
        
        notes.append({
            "start_time": current_time,
            "duration": beat_duration,
            "midi_note": midi_note,
            "velocity": velocity,
            "bol": bol
        })
        
        current_time += beat_duration
    
    return notes

# Usage:
tabla_notes = generate_tintal_pattern(
    start_time=180,  # Minute 3
    duration=300,    # 5 minutes
    tempo_bpm=110
)
```

### Add Validation Framework (20 minutes)

```python
def validate_composition(composition):
    """Check composition quality."""
    issues = []
    metrics = {}
    
    # Count total notes
    total_notes = sum(
        len(inst.get('notes', []))
        for inst in composition.get('instruments', [])
    )
    duration = composition['metadata']['duration_minutes']
    notes_per_min = total_notes / duration
    
    metrics['total_notes'] = total_notes
    metrics['notes_per_minute'] = notes_per_min
    
    if notes_per_min < 60:
        issues.append(f"❌ Low density: {notes_per_min:.0f} notes/min (target: 80+)")
    else:
        issues.append(f"✅ Good density: {notes_per_min:.0f} notes/min")
    
    # Check ornamentation
    ornamented = 0
    for inst in composition['instruments']:
        for note in inst.get('notes', []):
            if 'pitch_bend' in note or 'cc_events' in note:
                ornamented += 1
    
    ornamentation_pct = (ornamented / total_notes * 100) if total_notes > 0 else 0
    metrics['ornamentation_pct'] = ornamentation_pct
    
    if ornamentation_pct < 30:
        issues.append(f"❌ Low ornamentation: {ornamentation_pct:.0f}% (target: 40%+)")
    else:
        issues.append(f"✅ Good ornamentation: {ornamentation_pct:.0f}%")
    
    # Check tanpura
    tanpura = next(
        (i for i in composition['instruments'] if i['name'] == 'Tanpura'),
        None
    )
    if not tanpura or len(tanpura.get('notes', [])) < 4:
        issues.append("❌ Weak tanpura (need 4+ continuous notes)")
    else:
        issues.append("✅ Strong tanpura")
    
    print("\nComposition Validation:")
    for issue in issues:
        print(issue)
    
    return metrics, issues
```

---

## Expected Improvements

### Before (Current)
```
✗ Notes: 315 (36.6/min)
✗ Ornamentation: 0%
✗ Instruments: Wrong (Tabla as Taiko)
✗ Structure: No raga/tala
Result: Sounds boring, not Indian
```

### After (With Changes)
```
✓ Notes: 800+ (100+/min)  [2.5x improvement]
✓ Ornamentation: 40-50%   [New feature]
✓ Instruments: Correct    [Fixed mapping]
✓ Structure: Proper raga/tala  [Authentic]
Result: Sounds authentically Indian
```

---

## Measurement Checklist

After implementing changes, verify:

**Quantitative:**
- [ ] Notes per minute: 80-150 (was: 36.6)
- [ ] Ornamentation: 40%+ (was: 0%)
- [ ] Pitch bend events: 100+ (was: 0)
- [ ] Note duration variety: 5+ different lengths (was: uniform)
- [ ] Tanpura notes: 4-5 continuous (was: 2 sporadic)
- [ ] Tabla patterns: Follow tala from minute 3 (was: no tala)

**Qualitative:**
- [ ] Sounds like Indian classical music
- [ ] Has characteristic glides and shakes
- [ ] Melody flows naturally (not mechanical)
- [ ] Rhythm is interesting and varied
- [ ] Not boring or sparse
- [ ] Could pass as professional composition

---

## Testing Samples

Use these prompts to test improvements:

### Test 1: Heroic Composition
```
Generate a classical Indian music composition representing the independence 
of India and freedom fighters. Use Raga Bhairav, Tintal, 8 minutes.
```

**Expected:**
- Heroic mood from Bhairav raga
- Military-style rhythm from Tintal
- Proper melody development
- Authentic ornamentation

### Test 2: Devotional
```
Generate a classical Indian music composition with a devotional, 
serene mood. Use Raga Yaman, Ektal, 10 minutes.
```

**Expected:**
- Peaceful, meditative mood
- Softer pace from Ektal
- Clear raga structure
- Smooth, flowing ornamentation

### Test 3: Complex
```
Generate a classical Indian music composition combining heroic (Raga Bhairav, 
minutes 0-4) and devotional (Raga Yaman, minutes 4-8) moods. Use Tintal.
```

**Expected:**
- Raga modulation (rare in Indian classical, but interesting)
- Clear structural change at minute 4
- Proper tala throughout
- Authentic character for each raga

---

## Troubleshooting

### Problem: "Still no ornamentation"
**Solution:** AI may not follow new prompt. Use post-processor function from Step 2.

### Problem: "Notes still too sparse"
**Solution:** Be explicit in user prompt: "Include 400-600 notes minimum. Use rapid passages."

### Problem: "Doesn't sound like Indian music"
**Solution:** Check raga validation. AI may be using notes outside raga. Use validation function.

### Problem: "Tabla sounds wrong"
**Solution:** Ensure tabla uses MIDI channel 9 and proper note mapping. Test with real tabla samples or SFZ files.

### Problem: "Tanpura has gaps"
**Solution:** Ensure all tanpura notes have same duration (520+ beats for 8-10 min piece).

---

## Next Steps

1. **This week:** Implement Steps 1-3 (30 min)
2. **Next week:** Add extended features from 1-hour plan
3. **This month:** A/B test system prompt variations, measure improvements
4. **Long-term:** Develop SFZ instruments for better sounds

---

## Success Indicators

You'll know it's working when:

- ✅ Output has 800+ notes (not 315)
- ✅ Notes include pitch bends and vibrato (not just MIDI notes)
- ✅ Composition sounds like authentic Indian classical music
- ✅ Rhythm is interesting and follows tala patterns
- ✅ Melody has characteristic glides and shakes
- ✅ Friends/colleagues recognize it as Indian music

---

**Time to first improvement: 30 minutes**  
**Time to substantial improvement: 2 hours**  
**Time to production-ready: 1 week of iteration**

---

*Quick Start Guide v1.0*  
*Last updated: 2025-01-18*  
*Based on analysis in ANALYSIS.md*
