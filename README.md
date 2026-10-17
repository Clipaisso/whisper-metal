# whisper-metal

Builds de **[whisper.cpp](https://github.com/ggml-org/whisper.cpp)** com o backend **Metal** para macOS (Apple Silicon e Intel), mantidos pela **[Clipaisso](https://clipaisso.com)** para uso no app **Clipaisso Desktop**.

> Este repositório **não** modifica o whisper.cpp — apenas o **compila** com o Metal habilitado e publica o binário pronto numa release, já que o projeto oficial não distribui esse build para macOS.

## Por que ele existe

O Clipaisso Desktop transcreve áudio localmente (ferramenta de legendas) chamando o programa `whisper-cli` do whisper.cpp. No Windows o app baixa o build oficial de CPU e o nosso [whisper-vulkan](https://github.com/Clipaisso/whisper-vulkan).

Para macOS, as releases oficiais do whisper.cpp só trazem o **`xcframework`** — a biblioteca para quem escreve app em Swift/Objective-C. O **programa de linha de comando não é publicado**, então não há o que baixar.

Este repositório compila esse programa com o **Metal**, a GPU da Apple: roda em qualquer Mac com Apple Silicon (M1 em diante) e nos Intel com GPU compatível com Metal, sem instalar nada além do próprio macOS.

## O que tem na release

A cada execução, o workflow publica o asset **`whisper-metal-bin-macos.zip`** na release de tag fixa **`whisper-metal-latest`** (a URL nunca muda; o conteúdo é substituído):

```
https://github.com/Clipaisso/whisper-metal/releases/download/whisper-metal-latest/whisper-metal-bin-macos.zip
```

O zip contém:

| Arquivo | O que é |
|---|---|
| `whisper-cli` | Binário **universal** (Apple Silicon + Intel), estático, com o Metal embutido. Só depende de frameworks do próprio macOS (Metal, Accelerate, Foundation). |
| `LICENSE` | A licença MIT do whisper.cpp |
| `VERSION.txt` | A tag e o commit do whisper.cpp usados no build |

O binário tem assinatura **ad-hoc** (gratuita, sem conta Apple). Como quem baixa é o Clipaisso, e não o navegador, o arquivo não recebe a "quarentena" do macOS e roda sem o aviso do Gatekeeper.

## Requisito de runtime

**macOS 13.3 ou mais novo.** A primeira transcrição num Mac leva alguns segundos a mais: o macOS compila os shaders do Metal uma vez e guarda em cache.

## Como (re)gerar o build

O build roda inteiramente nos runners do GitHub — nada precisa ser instalado localmente.

1. Aba **Actions** → **"Build whisper.cpp (Metal)"** → **Run workflow**.
2. Informe a tag do whisper.cpp em `whisper_ref` (ex.: `b5454`). Use a mesma build do motor em CPU do app (`WHISPER_CPU_TAG`, em `src-tauri/src/whisper/mod.rs` do Clipaisso Desktop), para os dois sistemas transcreverem igual.
3. Ao terminar, a release `whisper-metal-latest` é atualizada com o novo zip.

> O whisper.cpp publica dois tipos de tag: as de versão (`v1.9.5`) e as de build (`b5454`). As duas servem aqui, porque o workflow compila o código-fonte, que existe em ambas.

Por baixo, o workflow (`.github/workflows/build.yml`):

- faz checkout do `ggml-org/whisper.cpp` na tag escolhida;
- compila o `whisper-cli` **duas vezes**, uma para `arm64` e outra para `x86_64`, e junta as duas com `lipo`. Compilar as duas arquiteturas de uma vez faz o ggml não reconhecer a CPU e usar o código genérico, cerca de **15x mais lento** transcrevendo em CPU;
- assina ad-hoc e confere que o binário não depende de nenhuma biblioteca de fora do macOS (uma do Homebrew funcionaria no runner e quebraria no Mac de quem usa);
- faz uma transcrição de teste com as mesmas flags que o Clipaisso passa (`-ojf -pp --dtw --vad --suppress-nst`);
- empacota e publica a release.

### Gerar num Mac, sem o GitHub Actions

Precisa só das Command Line Tools da Apple (`xcode-select --install`) e do [CMake](https://cmake.org/download/):

```sh
git clone --depth 1 --branch b5454 https://github.com/ggml-org/whisper.cpp.git
cd whisper.cpp
for arch in arm64 x86_64; do
  cmake -S . -B "build-$arch" -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_OSX_ARCHITECTURES="$arch" -DCMAKE_OSX_DEPLOYMENT_TARGET=13.3 \
    -DBUILD_SHARED_LIBS=OFF -DGGML_NATIVE=OFF -DGGML_OPENMP=OFF \
    -DGGML_METAL=ON -DGGML_METAL_EMBED_LIBRARY=ON -DGGML_METAL_USE_BF16=ON \
    -DGGML_CCACHE=OFF -DWHISPER_BUILD_TESTS=OFF -DWHISPER_BUILD_SERVER=OFF
  cmake --build "build-$arch" --config Release -j --target whisper-cli
done
mkdir -p pack
lipo -create build-arm64/bin/whisper-cli build-x86_64/bin/whisper-cli -output pack/whisper-cli
codesign --force --sign - pack/whisper-cli
cp LICENSE pack/
(cd pack && zip -X -r ../whisper-metal-bin-macos.zip .)
```

Depois é só subir o `whisper-metal-bin-macos.zip` na release `whisper-metal-latest`.

## Licença e créditos

O whisper.cpp é de autoria da **ggml-org** sob licença **MIT** — todo o código compilado aqui pertence a esse projeto. Este repositório apenas redistribui builds binários por conveniência. Consulte a licença original em [ggml-org/whisper.cpp](https://github.com/ggml-org/whisper.cpp/blob/master/LICENSE).

---

Mantido por **[Clipaisso](https://clipaisso.com)**.
