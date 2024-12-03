+++
title = "rustdesk 音频 源码解析"
date = 2024-08-19

[taxonomies]
tags = ["rustdesk", "rust"]
+++

rustdesk 是一个开源的远程桌面控制器。这是对受控端和控制端音频功能的解析。

<!-- more -->

# 目标

在控制端能够接受到受控端的实时音频，并播放。

- 接受受控端的音频数据
- 播放音频

# 业务流程

- 在控制端实时获取音频数据
- 将获取的音频数据发送给网络
- 获取从网络拿到的音频数据
- 播放音频数据

# 控制端的音频服务

audio service 实现了 1,2 功能。
该功能涉及到的库如下:

- opus 音频数据编码器
- 其他

```rust
impl super::service::Reset for State {
    fn reset(&mut self) {
        self.stream.take();
    }
}

fn run_restart(sp: EmptyExtraFieldService, state: &mut State) -> ResultType<()> {
    state.reset();
    sp.snapshot(|_sps: ServiceSwap<_>| Ok(()))?;
    match &state.stream {
        None => {
            state.stream = Some(play(&sp)?);
        }
        _ => {}
    }
    if let Some((_, format)) = &state.stream {
        sp.send_shared(format.clone());
    }
    RESTARTING.store(false, Ordering::SeqCst);
    Ok(())
}

fn run_serv_snapshot(sp: EmptyExtraFieldService, state: &mut State) -> ResultType<()> {
    sp.snapshot(|sps| {
        match &state.stream {
            None => {
                //在控制端实时获取音频数据
                state.stream = Some(play(&sp)?);
            }
            _ => {}
        }
        //将获取的音频数据发送给网络
        if let Some((_, format)) = &state.stream {
            sps.send_shared(format.clone());
        }
        Ok(())
    })?;
    Ok(())
}

//音频线程的主逻辑
pub fn run(sp: EmptyExtraFieldService, state: &mut State) -> ResultType<()> {
    if !RESTARTING.load(Ordering::SeqCst) {
        //每一个帧执行的逻辑
        run_serv_snapshot(sp, state)
    } else {
        //线程重置
        run_restart(sp, state)
    }
}
```
这里主要关注play的实现。
```rust
   fn play(sp: &GenericService) -> ResultType<(Box<dyn StreamTrait>, Arc<Message>)> {
        use cpal::SampleFormat::*;
        //获取当前设备
        let (device, config) = get_device()?;
        let sp = sp.clone();
        // Sample rate must be one of 8000, 12000, 16000, 24000, or 48000.
        let sample_rate_0 = config.sample_rate().0;
        let sample_rate = if sample_rate_0 < 12000 {
            8000
        } else if sample_rate_0 < 16000 {
            12000
        } else if sample_rate_0 < 24000 {
            16000
        } else if sample_rate_0 < 48000 {
            24000
        } else {
            48000
        };
        let ch = if config.channels() > 1 { Stereo } else { Mono };
        //构建流
        let stream = match config.sample_format() {
            I8 => build_input_stream::<i8>(device, &config, sp, sample_rate, ch)?,
            I16 => build_input_stream::<i16>(device, &config, sp, sample_rate, ch)?,
            I32 => build_input_stream::<i32>(device, &config, sp, sample_rate, ch)?,
            I64 => build_input_stream::<i64>(device, &config, sp, sample_rate, ch)?,
            U8 => build_input_stream::<u8>(device, &config, sp, sample_rate, ch)?,
            U16 => build_input_stream::<u16>(device, &config, sp, sample_rate, ch)?,
            U32 => build_input_stream::<u32>(device, &config, sp, sample_rate, ch)?,
            U64 => build_input_stream::<u64>(device, &config, sp, sample_rate, ch)?,
            F32 => build_input_stream::<f32>(device, &config, sp, sample_rate, ch)?,
            F64 => build_input_stream::<f64>(device, &config, sp, sample_rate, ch)?,
            f => bail!("unsupported audio format: {:?}", f),
        };
        //录制
        stream.play()?;
        Ok((
            Box::new(stream),
            Arc::new(create_format_msg(sample_rate, ch as _)),
        ))
    }
```
这里需要关注的是build_input_stream的实现。代码如下:
```rust
fn build_input_stream<T>(
        device: cpal::Device,
        config: &cpal::SupportedStreamConfig,
        sp: GenericService,
        sample_rate: u32,
        encode_channel: magnum_opus::Channels,
    ) -> ResultType<cpal::Stream>
    where
        T: cpal::SizedSample + dasp::sample::ToSample<f32>,
    {
        let err_fn = move |err| {
            // too many UnknownErrno, will improve later
            log::trace!("an error occurred on stream: {}", err);
        };
        let sample_rate_0 = config.sample_rate().0;
        log::debug!("Audio sample rate : {}", sample_rate);
        unsafe {
            AUDIO_ZERO_COUNT = 0;
        }
        let device_channel = config.channels();
        let mut encoder = Encoder::new(sample_rate, encode_channel, LowDelay)?;
        // https://www.opus-codec.org/docs/html_api/group__opusencoder.html#gace941e4ef26ed844879fde342ffbe546
        // https://chromium.googlesource.com/chromium/deps/opus/+/1.1.1/include/opus.h
        // Do not set `frame_size = sample_rate as usize / 100;`
        // Because we find `sample_rate as usize / 100` will cause encoder error in `encoder.encode_vec_float()` sometimes.
        // https://github.com/xiph/opus/blob/2554a89e02c7fc30a980b4f7e635ceae1ecba5d6/src/opus_encoder.c#L725
        let frame_size = sample_rate_0 as usize / 100; // 10 ms
        let encode_len = frame_size * encode_channel as usize;
        let rechannel_len = encode_len * device_channel as usize / encode_channel as usize;
        INPUT_BUFFER.lock().unwrap().clear();
        let timeout = None;
        let stream_config = StreamConfig {
            channels: device_channel,
            sample_rate: config.sample_rate(),
            buffer_size: BufferSize::Default,
        };
        let stream = device.build_input_stream(
            &stream_config,
            move |data: &[T], _: &InputCallbackInfo| {
                let buffer: Vec<f32> = data.iter().map(|s| T::to_sample(*s)).collect();
                let mut lock = INPUT_BUFFER.lock().unwrap();
                lock.extend(buffer);
                while lock.len() >= rechannel_len {
                    let frame: Vec<f32> = lock.drain(0..rechannel_len).collect();
                    send(
                        frame,
                        sample_rate_0,
                        sample_rate,
                        device_channel,
                        encode_channel as _,
                        &mut encoder,
                        &sp,
                    );
                }
            },
            err_fn,
            timeout,
        )?;
        Ok(stream)
    }
```
这里有两个对象要注意，一个是本地设备的采样率，一个是编码器的采样率。所传输的数据是编码器的音频数据。