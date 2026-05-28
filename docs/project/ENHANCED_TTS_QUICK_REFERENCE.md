# Enhanced TTS - Quick Reference Card

## 🚀 **QUICK START**

```dart
// 1. Get service
final tts = ref.watch(enhancedTtsServiceProvider);

// 2. Initialize
await tts.initialize();

// 3. Set voice
await tts.setVoice(locale: 'en-IN', gender: 'female');

// 4. Speak
await tts.speakAndWait('Hello!');
```

---

## 📚 **COMMON PATTERNS**

### **Interview Question**

```dart
await _enhancedTts.speakAndWait(
  question.question,
  locale: config.voiceAccent.localeCode,
  gender: config.aiPersonality.isFemale ? 'female' : 'male',
  rate: config.speechRate,
);
```

### **Greeting Message**

```dart
await _enhancedTts.speakAndWait(
  'Hello $userName! Welcome to your interview.',
  locale: 'en-IN',
  gender: 'female',
  rate: 0.9,
);
```

### **Long Text**

```dart
// Automatically chunked and preprocessed
await _enhancedTts.speakAndWait(
  longFeedbackText,
  locale: 'en-IN',
);
```

---

## 🎯 **VOICE SELECTION**

### **By Locale**

```dart
await tts.setVoice(locale: 'en-IN');  // Indian English
await tts.setVoice(locale: 'en-US');  // US English
await tts.setVoice(locale: 'en-GB');  // British English
```

### **By Gender**

```dart
await tts.setVoice(locale: 'en-IN', gender: 'male');
await tts.setVoice(locale: 'en-IN', gender: 'female');
```

### **By Name**

```dart
await tts.setVoice(
  locale: 'en-IN',
  preferredVoiceName: 'Google en-IN Female',
);
```

---

## ⚙️ **CONFIGURATION**

### **Update Settings**

```dart
await tts.updateConfig(TtsConfig(
  speechRate: 0.9,   // 0.5 to 2.0
  pitch: 1.0,        // 0.5 to 2.0
  volume: 1.0,       // 0.0 to 1.0
));
```

### **Temporary Override**

```dart
await tts.speak(
  'Fast message',
  rate: 1.5,  // Override default
  pitch: 1.1,
);
```

---

## 🛠️ **UTILITIES**

### **Preprocess Text**

```dart
final processed = TextPreprocessor.preprocess(rawText);
// Expands abbreviations, normalizes numbers, removes markdown
```

### **Split into Chunks**

```dart
final chunks = TextPreprocessor.splitIntoChunks(longText);
// Returns list of < 200 char chunks
```

---

## 🔍 **VOICE INSPECTION**

### **Get Available Voices**

```dart
final voices = await ref.read(availableVoicesProvider.future);
for (final voice in voices) {
  print('${voice.name} (${voice.locale}) - ${voice.quality.name}');
}
```

### **Get Current Voice**

```dart
final current = ref.watch(currentVoiceProvider);
print('Using: ${current?.name}');
```

### **Check Speaking State**

```dart
final isSpeaking = ref.watch(isSpeakingProvider);
if (isSpeaking) {
  await tts.stop();
}
```

---

## 🎨 **UI INTEGRATION**

### **Voice Selector**

```dart
final voices = ref.watch(availableVoicesProvider);

voices.when(
  data: (list) => DropdownButton<TtsVoice>(
    items: list.map((v) => DropdownMenuItem(
      value: v,
      child: Text('${v.name} (${v.quality.name})'),
    )).toList(),
    onChanged: (voice) async {
      await tts.setVoice(
        locale: voice!.locale,
        preferredVoiceName: voice.name,
      );
    },
  ),
  loading: () => CircularProgressIndicator(),
  error: (e, _) => Text('Error: $e'),
);
```

### **Speech Rate Slider**

```dart
final config = ref.watch(ttsConfigProvider);

Slider(
  value: config.speechRate,
  min: 0.5,
  max: 2.0,
  onChanged: (value) {
    ref.read(ttsConfigProvider.notifier).updateSpeechRate(value);
    tts.updateConfig(config.copyWith(speechRate: value));
  },
);
```

---

## 🐛 **ERROR HANDLING**

### **Try-Catch Pattern**

```dart
try {
  await tts.speakAndWait(text);
} catch (e) {
  debugPrint('TTS error: $e');
  // Show user-friendly message
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text('Speech unavailable')),
  );
}
```

### **Check Initialization**

```dart
if (!tts.isInitialized) {
  final success = await tts.initialize();
  if (!success) {
    // Handle initialization failure
    return;
  }
}
```

---

## 📊 **QUALITY TIPS**

### **Optimal Settings**

| Platform | Rate | Pitch | Volume |
|----------|------|-------|--------|
| Android | 0.9 | 1.0 | 1.0 |
| iOS | 0.85 | 1.0 | 1.0 |

### **Text Preprocessing**

✅ **DO**: Use TextPreprocessor for all TTS text  
✅ **DO**: Remove markdown before speaking  
✅ **DO**: Expand abbreviations (API → A P I)  
❌ **DON'T**: Send raw markdown to TTS  
❌ **DON'T**: Send very long text without chunking  

### **Voice Selection**

✅ **DO**: Set voice before speaking  
✅ **DO**: Prefer offline voices  
✅ **DO**: Match locale to user preference  
❌ **DON'T**: Change voice mid-sentence  
❌ **DON'T**: Assume voice availability  

---

## 🔧 **TROUBLESHOOTING**

### **No Sound**

```dart
// 1. Check volume
await tts.updateConfig(config.copyWith(volume: 1.0));

// 2. Check initialization
if (!tts.isInitialized) await tts.initialize();

// 3. Check speaking state
if (tts.isSpeaking) await tts.stop();

// 4. Try again
await tts.speak('Test');
```

### **Robotic Voice**

```dart
// 1. Check voice quality
final voice = tts.currentVoice;
if (voice?.quality == VoiceQuality.low) {
  // Select better voice
  await tts.setVoice(locale: 'en-IN');
}

// 2. Adjust rate
await tts.updateConfig(config.copyWith(speechRate: 0.9));
```

### **Voice Not Found**

```dart
// 1. List available voices
final voices = tts.availableVoices;
debugPrint('Available: ${voices.map((v) => v.name).join(", ")}');

// 2. Use fallback
await tts.setVoice(locale: 'en-US');  // Widely available
```

---

## 📱 **PLATFORM NOTES**

### **Android**

- Google TTS recommended
- Install from Play Store if missing
- Samsung TTS on Samsung devices
- System TTS as fallback

### **iOS**

- Siri voices preferred
- Enhanced quality voices best
- Download voices in Settings > Accessibility
- System voices always available

---

## 🎯 **BEST PRACTICES**

1. **Always initialize before use**
2. **Set voice before speaking**
3. **Use speakAndWait for sequential speech**
4. **Preprocess text for clarity**
5. **Handle errors gracefully**
6. **Stop speech when navigating away**
7. **Persist user preferences**
8. **Test on real devices**

---

## 📚 **DOCUMENTATION**

- **Implementation**: `ENHANCED_TTS_IMPLEMENTATION.md`
- **Examples**: `ENHANCED_TTS_EXAMPLES.md`
- **Summary**: `ENHANCED_TTS_SUMMARY.md`
- **Checklist**: `ENHANCED_TTS_CHECKLIST.md`

---

## 🆘 **SUPPORT**

### **Common Issues**

| Issue | Solution |
|-------|----------|
| No voices available | Install Google TTS (Android) |
| Robotic sound | Select enhanced voice |
| Too fast | Reduce speech rate to 0.85 |
| Too slow | Increase speech rate to 1.0 |
| Wrong language | Set correct locale |
| No sound | Check volume, initialization |

### **Debug Logging**

```dart
// Enable in enhanced_tts_service.dart
debugPrint('TTS: Voice selected: ${voice.name}');
debugPrint('TTS: Quality score: ${voice.qualityScore}');
debugPrint('TTS: Speaking: $text');
```

---

**Quick Reference v1.0.0** | Last Updated: 2026-02-05
