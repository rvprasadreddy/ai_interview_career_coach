# Enhanced TTS Integration Example

## Quick Start

### 1. Basic Usage

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/providers/tts_providers.dart';

class InterviewScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final ttsService = ref.watch(enhancedTtsServiceProvider);
    final isSpeaking = ref.watch(isSpeakingProvider);
    
    return Scaffold(
      body: Column(
        children: [
          Text(isSpeaking ? 'Speaking...' : 'Ready'),
          ElevatedButton(
            onPressed: () async {
              await ttsService.speakAndWait(
                'Hello! Welcome to your AI interview.',
                locale: 'en-IN',
                gender: 'female',
              );
            },
            child: Text('Start Interview'),
          ),
        ],
      ),
    );
  }
}
```

### 2. Integration with Interview Provider

```dart
// In interview_provider.dart

import '../providers/tts_providers.dart';

class InterviewController extends StateNotifier<InterviewState> {
  final EnhancedTtsService _enhancedTts;
  
  InterviewController(this._enhancedTts) : super(const InterviewState());
  
  Future<void> askQuestion(String question) async {
    // Set voice based on personality
    await _enhancedTts.setVoice(
      locale: state.config.voiceAccent.localeCode,
      gender: state.config.aiPersonality.isFemale ? 'female' : 'male',
    );
    
    // Speak with preprocessing
    await _enhancedTts.speakAndWait(
      question,
      rate: state.config.speechRate,
    );
  }
}

// Provider
final interviewControllerProvider = 
    StateNotifierProvider<InterviewController, InterviewState>((ref) {
  return InterviewController(
    ref.watch(enhancedTtsServiceProvider),
  );
});
```

### 3. Voice Settings Screen

```dart
class VoiceSettingsScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final availableVoices = ref.watch(availableVoicesProvider);
    final currentVoice = ref.watch(currentVoiceProvider);
    final config = ref.watch(ttsConfigProvider);
    final ttsService = ref.watch(enhancedTtsServiceProvider);
    
    return Scaffold(
      appBar: AppBar(title: Text('Voice Settings')),
      body: availableVoices.when(
        data: (voices) => ListView(
          children: [
            // Current voice display
            ListTile(
              title: Text('Current Voice'),
              subtitle: Text(currentVoice?.name ?? 'Default'),
            ),
            
            // Voice list
            ...voices.map((voice) => ListTile(
              title: Text(voice.name),
              subtitle: Text('${voice.locale} • ${voice.quality.name}'),
              trailing: voice == currentVoice 
                  ? Icon(Icons.check, color: Colors.green)
                  : null,
              onTap: () async {
                await ttsService.setVoice(
                  locale: voice.locale,
                  preferredVoiceName: voice.name,
                );
              },
            )),
            
            // Speech rate slider
            ListTile(
              title: Text('Speech Rate: ${config.speechRate.toStringAsFixed(2)}'),
              subtitle: Slider(
                value: config.speechRate,
                min: 0.5,
                max: 2.0,
                onChanged: (value) {
                  ref.read(ttsConfigProvider.notifier).updateSpeechRate(value);
                  ttsService.updateConfig(config.copyWith(speechRate: value));
                },
              ),
            ),
            
            // Test button
            ElevatedButton(
              onPressed: () async {
                await ttsService.speak(
                  'This is a test of the text to speech system.',
                );
              },
              child: Text('Test Voice'),
            ),
          ],
        ),
        loading: () => Center(child: CircularProgressIndicator()),
        error: (err, stack) => Center(child: Text('Error: $err')),
      ),
    );
  }
}
```

### 4. Migration from Legacy Service

```dart
// BEFORE (speech_service.dart)
await _speechService.speakAndWait(
  greeting, 
  rate: state.config.speechRate,
  language: locale,
  isFemale: state.config.aiPersonality.isFemale,
);

// AFTER (enhanced_tts_service.dart)
await _enhancedTts.speakAndWait(
  greeting,
  locale: locale,
  gender: state.config.aiPersonality.isFemale ? 'female' : 'male',
  rate: state.config.speechRate,
);
```

## Testing

```dart
void main() {
  testWidgets('TTS speaks correctly', (tester) async {
    final container = ProviderContainer();
    final ttsService = container.read(enhancedTtsServiceProvider);
    
    await ttsService.initialize();
    
    expect(ttsService.isInitialized, true);
    expect(ttsService.availableVoices.isNotEmpty, true);
    
    await ttsService.speak('Test message');
    
    // Verify speaking state
    expect(ttsService.isSpeaking, true);
  });
}
```

## Best Practices

1. **Always initialize before use**
   ```dart
   await ttsService.initialize();
   ```

2. **Set voice before speaking**
   ```dart
   await ttsService.setVoice(locale: 'en-IN', gender: 'female');
   await ttsService.speak('Hello');
   ```

3. **Use speakAndWait for sequential speech**
   ```dart
   await ttsService.speakAndWait('First message');
   await ttsService.speakAndWait('Second message');
   ```

4. **Stop speech when navigating away**
   ```dart
   @override
   void dispose() {
     ref.read(enhancedTtsServiceProvider).stop();
     super.dispose();
   }
   ```

5. **Handle errors gracefully**
   ```dart
   try {
     await ttsService.speak('Message');
   } catch (e) {
     debugPrint('TTS error: $e');
     // Show user-friendly message
   }
   ```
