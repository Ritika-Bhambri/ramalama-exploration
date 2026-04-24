# ramalama-exploration

RamaLama is an open-source developer tool that simplifies the local serving of AI models from any source and facilitates their use for inference in  production through containers also learn how ramalama makes working with AI boring

In this experiment I have 
- Installed Ramalama
- Pulled different models through differnt transports
- I tested each model with two standard benchmark questions "List the Four Foundations of the Fedora project" and to evaluate their accuracy and "Write a simple RPM spec file header (Name, Version, Release, Summary, License) for a package named 'test-app'. Use standard RPM spec syntax."
- I identify where the models provide correct answers against the benchmarks or 'hallucinate'

## Pre- requisites

### Installing Podman
<img width="1462" height="430" alt="ramalama-podman-install" src="https://github.com/user-attachments/assets/e7f93e58-f6e1-4720-8a46-171bffe00332" />

### Verifying Podman

Command used - `podman run hello-world`

<img width="797" height="423" alt="ramalama-podma-run" src="https://github.com/user-attachments/assets/35e9dd60-c125-4507-b253-49cdb5049e36" />

### System Memory Check
<img width="800" height="267" alt="ramalama-ram" src="https://github.com/user-attachments/assets/92393baf-5102-4db0-8027-41bcea96e227" />


## Installing Ramalama

<img width="847" height="422" alt="ramalama-install-main" src="https://github.com/user-attachments/assets/61f6d045-fb10-4dba-9a39-ba904be3842d" />

## Version check

<img width="822" height="423" alt="ramalama-version-main" src="https://github.com/user-attachments/assets/01068855-b7af-4cf9-870c-26533fea361a" />

## Qwen via Huggingface

### Pulling the model

`ramalama pull hf://Qwen/Qwen2.5-0.5B-Instruct-GGUF` 

Output

<img width="1396" height="222" alt="Qwen-via-Hf" src="https://github.com/user-attachments/assets/0d3441ec-d611-4b01-be7f-deaf1a5f2346" />

### Benchmark Test 1

`ramalama run hf://Qwen/Qwen2.5-0.5B-Instruct-GGUF "List the Four Foundations of the Fedora project.`

Output

<img width="1482" height="175" alt="Qwen-response1" src="https://github.com/user-attachments/assets/49420abd-ecb8-41a1-8c6b-5e667f0ec938" />


Qwen straight up refused to answer the question over safety concerns. Small models are often "over-aligned" during their safety training. The model likely flagged the word "Foundations" or "Fedora" as a potential request for sensitive, internal, or proprietary information.

Because the model is only 0.5B parameters, it lacks the nuance to realize that the "Four Foundations of Fedora" is public, community knowledge. It chose to play it safe and refused to give an answer rather than risk giving a "harmful" answer.


### Benchmark Test 2

`ramalama run hf://Qwen/Qwen2.5-0.5B-Instruct-GGUF "Write a simple RPM spec file header (Name, Version, Release, Summary, License) for a package named 'test-app'."`

Output

<img width="1337" height="557" alt="Qwen-response3" src="https://github.com/user-attachments/assets/e41e3127-4344-4d31-b037-c0ee284ab246" />


I could notice 3 failures in this response

-The model wrapped the output in ```xml tags. This is a Syntax Failure. RPM spec files are plain text, and adding XML tags would cause rpmbuild to fail immediately.

-In the BuildRequires section it repeated the same line seven times.

-RPM specs don't use Author: or Author-email:. They use Packager: (optionally), but the info is usually handled via the %changelog.


## Granite via Ollama

`ramalama pull ollama://granite3.1-dense:2b`

<img width="1406" height="263" alt="granite-via-ollama" src="https://github.com/user-attachments/assets/294ce3c6-3c1f-4264-96d7-b66e5d4039cf" />



**Output** 


I was able to pull granite via ollama but when I ran benchmarch test 1 it stopped after a the 180-second health check timeout. 


<img width="1442" height="421" alt="granite-failure" src="https://github.com/user-attachments/assets/d9b362eb-48ff-455c-b63f-f01e68a8912e" />

### Reason for the Error

Granite 3.1 2B is a "Dense" model. Unlike a 0.5B model, it has 2.5 billion parameters that must be uncompressed and loaded into RAM. On my system, this process took longer than the 3-minute limit set by RamaLama.

Here is how the memory looked like after pulling granite and trying to run the benchmark 1 test

<img width="741" height="156" alt="memory-after-granite" src="https://github.com/user-attachments/assets/4c1222bc-1077-4f3a-99cd-d5ae8e945818" />

## Granite 3 MoE1B Model via ollama

Since Granite 3.1 2B would not run on my hardware as a workaround I pulled the smaller Granite 3 MoE1B model. IBM has MoE (Mixture of Experts) versions. These are smarter than Qwen but lighter on the CPU because they only activate a fraction of their parameters at a time.

`ramalama pull ollama://granite3-moe:1b` 

Output

<img width="1303" height="165" alt="Granite-MOE" src="https://github.com/user-attachments/assets/6152ca27-1e9c-4734-a394-9346b8927b02" />

### Benchmark Test 1

`ramalama run ollama://granite3-moe:1b "List the Four Foundations of the Fedora project."`

Output

<img width="1308" height="437" alt="Granite-MOE-response1" src="https://github.com/user-attachments/assets/0f145585-71c1-4c78-b9ea-2fd6e66a0c7d" />



While unlike Qwen that refused to answer the question, Granite 3 MoE1B answered the question but it hallucinated the core identity of the Fedora Project. It replaced the foundations (Friends, Freedom, Features, First) with generic corporate/social values.

### Benchmark Test 2

`ramalama run ollama://granite3-moe:1b "Write a simple RPM spec file header (Name, Version, Release, Summary, License) for a package named 'test-app'. Use standard RPM spec syntax."`

Output

<img width="1311" height="426" alt="Granite-MOE-response3" src="https://github.com/user-attachments/assets/7837eb5f-0fb9-4fc9-b752-d4b19e2f3bc1" />


- This test resulted in cross platform confusion. Instead of an RPM spec file the model provided a Debian/Ubuntu control file syntax (using keywords like Package:, Architecture:, and Description:) and mixed it with a wrong defination `noarch`.
- I noticed that model claimed that `noarch` architecture keyword means that the package is multimedia content, not related to the Linux kernel. 
- The Truth is `noarch` signifies "No Architecture". It means that the package is platform-independent and can run on x86, ARM, or any other CPU.

## Tinyllama via Ollama

`ramalama pull ollama://tinyllama`

<img width="1382" height="257" alt="tinylama-pull" src="https://github.com/user-attachments/assets/ab9431d5-3de6-469b-8bd0-4cc259f5a9b5" />

### Benchmark Test 1 

`ramalama run ollama://tinyllama "What are the Four Foundations of the Fedora project?"`

Output

<img width="1301" height="371" alt="Tinyllama-response1" src="https://github.com/user-attachments/assets/de4eb3a6-cf35-429d-9e1f-c0cead01f913" />


- This also resulted in hallucination.
- While the model's response is not accurate unlike Granite 3 MoE1B that threw around some corporate talk, TinyLlama guessed around the word 'Fedora', it's association with Linux. Since every Linux distribution needs packages and manuals, it assumed these must be the foundations.
- The only thing partially correct was that it gave **Community** which is close in spirit to the actual fedora foundation-**Friends** 


### Benchmark Test 2 

`ramalama run ollama://tinyllama "Write a simple RPM spec file header (Name, Version, Release, Summary, License) for a package named 'test-app'. Use standard RPM spec syntax."`

Output

<img width="1307" height="172" alt="Tinyllama-response2" src="https://github.com/user-attachments/assets/e5f57d36-6214-4cf5-a0ec-1335a45f2808" />

- The model asked for a "Depends" section.
- This is a sign that TinyLlama confused Debian/Ubuntu (.deb) syntax with Fedora (.rpm) syntax. The dependency tag `Depends:` is used in the Debian ecosysyem whereas Fedora uses `Requires:`  tag.


## TinyLlama via Huggingface

Pulling the model via huggingface
```python
time ramalama pull hf://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF
```

<img width="1410" height="262" alt="tinyllama-huggingface" src="https://github.com/user-attachments/assets/8bd48655-311c-4acc-8f30-2fd1c49b9585" />

### Benchmark Test 1

```python
ramalama run hf://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF "What are the Four Foundations of the Fedora project?"
```

**Output**

<img width="1477" height="226" alt="image" src="https://github.com/user-attachments/assets/c07e8107-cfbe-4758-b648-121413d42ac7" />



- The response from this model about the four foundations of the Fedora project also hallucinated. It starts with naming the four foundations as open source software, architecture, kernel and community. It guesses around the word Fedora and comes up with a few technical terms that can be associated with Fedora.  
- It goes completly off the rails at the end by mentioning that the names of these foundations are based on the principles of sustainable forestry, which are based on the principles of organic and regenerative agriculture.


### Benchmark Test 2 

```python
ramalama run hf://TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF "Write a simple RPM spec file header (Name, Version, Release, Summary, License) for a package named 'test-app'. Use standard RPM spec syntax."
```

Output

<img width="1468" height="598" alt="image" src="https://github.com/user-attachments/assets/92f942dc-3ff9-4b8c-80bf-b29ba19f9355" />


- TinyLlama did not produce valid syntax for RPM Spec.  Instead of standard RPM tags, it used bracketed placeholders like [Summary].
- The model also simulated a conversation between a user and an assistant. It  hallucinated a new user prompt (<|user|> Can you please add some information....)

## OCI Transport

OCI (Open Container Initiative) is the standard format used by Docker and Podman for container images. RamaLama supports it as a transport, which means models can be stored and distributed through any container registry. OCI pulls work in layers copying blobs, verifying a config, and writing a manifest.

```python
time ramalama pull oci://quay.io/ramalama/tinyllama
```

<img width="1053" height="213" alt="image" src="https://github.com/user-attachments/assets/c2779d4f-168e-42db-9131-5b7b375aad8b" />

### Analysing the error

The error `quay.io/ramalama/tinyllama:latest does not exist` is a 404 - the image path is not published on quay.io. The time output shows it failed in 2.395 seconds. RamaLama checked the registry, got a not-found response, and exited immediately. 

**Finding:**

OCI registries like quay.io do not have a standardized public model namespace the way Ollama does. On Ollama, any model listed at ollama.com/library is pullable with no setup. On quay.io, models must be explicitly published by their maintainers as OCI artifacts. There is no central catalog to browse. This makes OCI transport less convenient for discovery and experimentation but more suitable for controlled production deployments


## Ramalama - serve 

All previous experiments used `ramalama run` which starts a model,accepts one prompt, returns one response. For this experiment I chose `ramalama serve`. It starts the model as an OpenAI-compatible REST API server. Any code that can call the OpenAI API can call a RamaLama served model, which means the RAG pipeline can be written in Python using standard libraries without any RamaLama-specific code.

### Terminal 1: Starting the Server


```bash
ramalama serve ollama://tinyllama
```

**Output**

<img width="1472" height="572" alt="Terminal1" src="https://github.com/user-attachments/assets/bdde1430-f83e-455b-b513-8fde20473b79" />



**Analysing few significant lines from the Output**

- `model params = 1.10 B` - confirms TinyLlama 1.1B loaded correctly
- `file type = Q4_0` - confirms quantization level
- `KV buffer size = 44.00 MiB` - total KV cache is only 44MB,far below the 6.1GB available. This is why TinyLlama runs whilegranite (which needed 10GB KV cache)     timed out.
- `n_slots = 4` - the server can handle 4 concurrent requests,not just one. This is essential for a RAG system where multiple
  users might query simultaneously.
- `server is listening on http://0.0.0.0:8080` — the model is now accessible as a REST API on port 8080
- `all slots are idle` - server is running and waiting for requests.

### Terminal 2: Querying via REST API

With the server running in Terminal 1, I opened a second terminaland sent an HTTP request using curl the same way a RAG pipeline would query the model programmatically.

**Command:**
```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tinyllama",
    "messages": [{"role": "user", "content": "What are the Four Foundations of the Fedora project?"}],
    "max_tokens": 300
  }' | python3 -m json.tool
```


**What this command does:**
- `http://localhost:8080/v1/chat/completions` - this is the exact same endpoint path used by the OpenAI API. Any code written for OpenAI works here without modification.
- `-H "Content-Type: application/json"` - standard JSON header
- The body follows OpenAI's chat completion format with `messages` array and `max_tokens`
- `python3 -m json.tool` - formats the raw JSON response for readability


**Output**

<img width="1318" height="592" alt="Terminal2" src="https://github.com/user-attachments/assets/1100fc01-5354-43ea-b98b-91c07d050e79" /> 

**Key fields from the response:**

```json
{
    "model": "tinyllama",
    "choices": [
        {
            "message": {
                "role": "assistant",
                "content": "[model response here]"
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 20,
        "completion_tokens": 85,
        "total_tokens": 105
    }
}
```

**What the response fields mean:**

- `finish_reason: "stop"` - the model completed naturally, not cut off by max_tokens. Clean termination.
- `prompt_tokens: 20` - the input question used 20 tokens
- `completion_tokens: 85` - the model generated 85 tokens
- `total_tokens: 105` - total context consumed this request

**Meanwhile in Terminal 1**, after the curl request was sent, the server logs showed:

slot  release: id 0 | task 1 | stop due to EOS
srv  update_slots: all slots are idle


<img width="1301" height="578" alt="Terminal1_After" src="https://github.com/user-attachments/assets/050b93fe-9995-450a-b0ac-fdd6f824e042" />

`stop due to EOS` confirms the model reached its natural end-of-sequence token - the response was complete. The server then returned to idle,
ready for the next request.
  


## Transport comparison

| Transport | Model | Pull Result | Run Result |
|---|---|---|---|
| HuggingFace | Qwen 0.5B | ✅ | ✅ (refused to answer) |
| Ollama | Granite dense 2B | ✅ | ❌ RAM timeout |
| Ollama | Granite MoE 1B | ✅ | ✅ (hallucinated) |
| Ollama | TinyLlama 1.1B | ✅ | ✅ (hallucinated) |
| HuggingFace | TinyLlama 1.1B | ✅ | ✅ (hallucinated) |
| OCI quay.io | TinyLlama | ❌ Does not exist | — |

## Failure mode analysis

Three distinct failure modes emerged - not one:

1. **Safety refusal** - Qwen refused to answer citing safety concerns. The model is too small to differenciate publically available knowledge from sensitive information.

2. **RAM timeout** - Granite dense 2B timed out. Disk size (1.46 GB) is not the same as runtime RAM required (~10 GB KV cache). Since there was no swap available it meant there was no fallback.

3. **Hallucination** - Every model that did run produced completely wrong answers on Fedora-specific questions. Granite MoE invented corporate values. TinyLlama invented Linux components and then connected Fedora to sustainable forestry.

## The controlled transport experiment

TinyLlama was pulled from both Ollama and HuggingFace. The model weights are identical. The responses to the Four Foundations question were structurally the same hallucination. This confirms that transport does not affect model output. The knowledge gap is in the weights and not in how they were delivered


 ## Does Ramalama make AI boring

Talking in terms of tooling and infrastructure, ramalama does make AI boring. The same `ramalama pull` and `ramalama run` command is used irrespective of whether the model comes from Hugging Face or Ollama. RamaLama takes care of the heavy lifting of containers (Podman/Docker) in the background. `ramalama serve`, a single command turns a local model into an OpenAI-compatible API without using any configuration files, port mapping or server setup. It takes away the hassle of setting up python environments CUDA drivers etc.



































