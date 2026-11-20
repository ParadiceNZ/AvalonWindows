
# AvalonWindows

A fork of [Avalon](https://github.com/freriadoc/Avalon) specifically fine-tuned for **Windows** on AMD Radeon GPUs.

Contains Windows-specific code paths focusing on doing everything possible to maximise both prompt reading and token generation speed.

(coming soon: a table showing a comparison to current upstream Avalon builds)



# About Avalon

Avalon is a fork of [llama.cpp](https://github.com/ggml-org/llama.cpp) built for one model, [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next) (512 experts, 10 active, 48 layers), on one consumer GPU with the experts in system RAM.

<p align="center"><img src="media/avalon-logo.png" alt="Avalon" width="200"></p>

## Credits

- [Avalon](https://github.com/freriadoc/Avalon) - the upstream these changes are based on.
- [llama.cpp](https://github.com/ggml-org/llama.cpp) and the ggml authors: the base.
- [ik_llama.cpp](https://github.com/ikawrakow/ik_llama.cpp) (Iwan Kawrakow): the IQ3_K and IQ3_KS types, the shared MTP head support.
- [Strata](https://github.com/Niko1221/Strata) (Niko1221): the expert ranking format and the preload idea.
- [Unsloth](https://huggingface.co/unsloth): the MTP head.
- [drluoto / Vitronia](https://huggingface.co/drluoto): the quantizations and the FR-Spec token list our GGUF files are built from.
- [Qwen](https://huggingface.co/Qwen): the model.


## License

MIT, as llama.cpp. The model files are under the Qwen Community License 1.0.
