# 05 - 音频系统 (Audio System)

Unity 的音频系统由 AudioSource（播放）、AudioListener（接收）、AudioClip（音频文件）三大核心组成。

## 核心组件

### AudioListener
- 挂在**摄像机**上，相当于"耳朵"
- 场景中**只能有一个**（多个会警告）

### AudioSource
- 挂在发声物体上，控制播放行为
- 常用属性：`clip`、`volume`、`pitch`、`loop`、`spatialBlend`

### AudioClip
- 音频文件本身（.wav、.mp3、.ogg 等）
- 导入设置中可选择压缩方式和加载方式

## 基础播放

```csharp
using UnityEngine;

public class AudioBasics : MonoBehaviour
{
    public AudioClip bgmClip;       // 背景音乐
    public AudioClip sfxClip;       // 音效
    private AudioSource audioSource;

    void Start()
    {
        audioSource = GetComponent<AudioSource>();
    }

    void Update()
    {
        // 播放音效（推荐用 PlayOneShot，不会打断当前播放）
        if (Input.GetKeyDown(KeyCode.Space))
        {
            audioSource.PlayOneShot(sfxClip);
        }

        // 播放/暂停/停止 BGM
        if (Input.GetKeyDown(KeyCode.B))
        {
            if (audioSource.isPlaying)
                audioSource.Pause();
            else
                audioSource.Play();
        }

        if (Input.GetKeyDown(KeyCode.S))
        {
            audioSource.Stop();
        }
    }

    // 播放一次性的音效（不挂 AudioSource 也能用）
    public void PlaySFX()
    {
        AudioSource.PlayClipAtPoint(sfxClip, transform.position);
    }
}
```

## 2D 音效 vs 3D 音效

```csharp
public class AudioSpatial : MonoBehaviour
{
    public AudioClip clip3D;
    private AudioSource source;

    void Start()
    {
        source = gameObject.AddComponent<AudioSource>();
        source.clip = clip3D;

        // spatialBlend: 0 = 纯2D, 1 = 纯3D
        source.spatialBlend = 1f; // 3D 音效

        // 3D 音效设置
        source.minDistance = 1f;   // 1米内音量最大
        source.maxDistance = 50f;  // 50米外听不到
        source.rolloffMode = AudioRolloffMode.Linear; // 线性衰减

        source.Play();
    }
}
```

**使用场景：**
- **2D 音效**：UI 音效、BGM（不受距离影响）
- **3D 音效**：环境音、脚步声、爆炸声（距离越远越小声）

## AudioMixer（混音器）

用于**分组控制音量**，比如分开调节 BGM、SFX、语音的音量。

### 创建 Mixer

1. **Window → Audio → Audio Mixer**
2. 创建 Groups（如 BGM、SFX、Voice）
3. 将 AudioSource 的 Output 指向对应的 Group

### 代码控制

```csharp
using UnityEngine;
using UnityEngine.Audio;

public class MixerControl : MonoBehaviour
{
    public AudioMixer audioMixer;

    // 设置音量（线性 0~1 转对数 dB）
    public void SetBGMVolume(float volume)
    {
        // volume 0~1 映射到 -80~0 dB
        float dB = volume > 0.001f ? Mathf.Log10(volume) * 20f : -80f;
        audioMixer.SetFloat("BGMVolume", dB);
    }

    public void SetSFXVolume(float volume)
    {
        float dB = volume > 0.001f ? Mathf.Log10(volume) * 20f : -80f;
        audioMixer.SetFloat("SFXVolume", dB);
    }

    // 静音
    public void MuteBGM(bool mute)
    {
        audioMixer.SetFloat("BGMVolume", mute ? -80f : 0f);
    }
}
```

### Snapshot（快照切换）

```csharp
public class MixerSnapshot : MonoBehaviour
{
    public AudioMixer mixer;
    public AudioMixerSnapshot normalSnapshot;
    public AudioMixerSnapshot pausedSnapshot;

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.P))
        {
            // 暂停时切换到模糊/低通滤波效果
            pausedSnapshot.TransitionTo(0.5f); // 0.5秒过渡
        }

        if (Input.GetKeyDown(KeyCode.R))
        {
            normalSnapshot.TransitionTo(0.5f);
        }
    }
}
```

## 音频池（对象池播放音效）

高频播放音效时，反复 `PlayOneShot` 会创建临时对象。用对象池优化：

```csharp
using System.Collections.Generic;
using UnityEngine;

public class AudioPool : MonoBehaviour
{
    public static AudioPool Instance;

    public int poolSize = 10;
    private Queue<AudioSource> pool = new Queue<AudioSource>();

    void Awake()
    {
        Instance = this;

        // 预创建 AudioSource 池
        for (int i = 0; i < poolSize; i++)
        {
            AudioSource source = gameObject.AddComponent<AudioSource>();
            source.playOnAwake = false;
            pool.Enqueue(source);
        }
    }

    public void Play(AudioClip clip, Vector3 position, float volume = 1f)
    {
        if (pool.Count == 0) return;

        AudioSource source = pool.Dequeue();
        source.clip = clip;
        source.volume = volume;
        source.transform.position = position;
        source.Play();

        // 播放完后回收
        StartCoroutine(Recycle(source, clip.length));
    }

    System.Collections.IEnumerator Recycle(AudioSource source, float delay)
    {
        yield return new WaitForSeconds(delay);
        pool.Enqueue(source);
    }
}

// 使用：AudioPool.Instance.Play(clip, transform.position);
```

## 实用技巧

| 技巧 | 说明 |
|------|------|
| `PlayOneShot` 替代 `Play` | 播放音效不会中断当前音频 |
| `AudioSource.PlayClipAtPoint` | 不需要挂 AudioSource 的一次性音效 |
| 预加载 `AudioClip.LoadAudioData()` | 避免播放时卡顿 |
| Vorbis 压缩 | 音乐用 Vorbis，体积小质量好 |
| PCM 无压缩 | 短音效用 PCM，加载快无解码开销 |
| `audioSource.time` | 获取/设置当前播放位置 |

## 注意事项

1. 场景中**只有一个 AudioListener**，多余的删掉
2. BGM 用 `loop = true`，音效不需要循环
3. 移动平台注意音频格式：**音乐用 Streaming，短音效用 DecompressOnLoad**
4. `pitch` 改变速率，`0.5` = 慢速，`2` = 快速（可用于子弹时间效果）
5. 3D 音效的衰减曲线要根据游戏类型调整
