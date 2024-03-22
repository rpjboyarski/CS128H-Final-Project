# CS 128 Honors Project

## Group
Ronan Boyarski - ronanpb2

## Project Introduction
The goal of this project is to implement the essential components for a Command and Control Framework in Rust. For the unfamiliar, a Command and Control (hereafter C2) Framework is a system designed to securely and covertly manage compromised computers in a way that makes further network exploitation fast, efficient, and stealthy. Some of the most popular frameworks at the moment include Cobalt Strike, Havoc, Sliver, PowerShell Empire, and Metasploit, although many of these are quite old. 

I've chosen to work on this project because this is a tool that I need to continue improving at offensive cybersecurity work, but the specific version that I need does not currently exist. My goal for the moment is just to implement the most basic and essential aspects of a C2 Framework, as it usually takes thousands of combined hours to create a complete one.

## Technical Overview
Creating a cross-platform, transport-layer agnostic method of secure communication across three parties is a surprising amount of work. For a C2 Framework, the infrastructure has three major parts: the client, team server, and payload.

### Client
The client is the application that the operator uses to manage compromised computers and further compromise a given network. For the moment, it is CLI based, although I plan on moving it to a GUI with Iced in the future. This is easily the simplest part of the application, as it provides the user with a command prompt, takes commands, and sends it to the server using Reqwest.

#### Client Tech Stack

HTTP Requests with Reqwest

### Team Server
While the team server isn't the most complicated aspect of a C2, it is certainly the most work. The team server is responsible for authentication & access control and managing all of the actions and data structures taken by the client (or multiple clients, if there are going to be many operators per team). This means that the team server must also include a database. Finally, the teamserver is designed to listen on localhost and operators should SSH tunnel in - it is not going to be internet exposed. The internet exposed portion of the team server is the **Listener**.

#### Listeners
Listeners are small subsections of a team server that are spawned by operators through the client. These are the internet-exposed endpoints that the malware payload must authenticate to and call back to. These are separate for a few reasons:
1. The listeners can be programmed to transport data differently every time one is deployed using a configuration file, and they can be hot-swapped at runtime to change how their routes are handled. This allows one team server to have many different modes of communication which can be changed at a moment's notice.
2. The listeners are separate from the team server, so an external user will have no visibility into the internal API for the team server and no chance of even being able to log into the management interface.
3. Listeners must be able to bind to different ports if they use non-http protocols, like encrypted TCP, or even something like websockets.

#### Server Tech Stack
1. Axum for the web server (there is no frontend)
2. PostgreSQL for the database
3. SQLX, ModQL & others for building queries
4. Transport between the client and the server will be encrypted with ChaCha20Poly1305 on top of HTTPS - this is because some solutions implement deep packet inspection, making HTTPS alone insecure for some targets.

### Payload
The payload is the heart of the project - the malware that is delivered to a compromised host. While Rust is cross platform in theory, in practice programming for cybersecurity is often highly platform-specific, meaning that this first iteration will target only Windows, although I am planning on adding support for Linux & MacOS later. As malware, it unfortunately is going to break many of the good design practices, including using FFI for syscalls and custom (unsafe) assembly code for reflectivly loading DLLs. The payload itself has four components:
1. The injector, which is a wrapper file to execute the rest of the content and is designed to be heavily modified by the end user. The reason is that it is the injector which is responsible for evading antivirus detection. For this project, because it is for academic purposes, the injector is designed to cause immediate detection so that the project cannot be directly copy-pasted for malicious purposes. However, in a real application, it would be trivial to modify the injector so that the rest of the payload is undetected.
2. The loader, which is custom assembly code written to load a DLL from memory (not from the disk!) without kernel callbacks detecting that a DLL has been loaded
3. The actual payload, which is a normal Rust project compiled into a DLL. By doing this, it's possible to use the entire standard library without having to write a file to disk, which is highly unusual for malware and provides a lot of features.

Example execution is as follows:
1. The injector is called, which loads the loader into memory and executes its entry point
2. The loader loads the payload into memory, calculates its base address and size, and calls the payload's entry point with those parameters. Otherwise, having a program calculate its own length and base address is impossible.
3. The payload executes, and authenticates to a listener deployed to the team server, establishing a secure connection
4. The operator is now free to interact with the payload to control the victim computer

If that seems complicated, I will provide a video walkthrough for the final project. It's actually much simpler doing it instead of reading it.

## Development Roadmap

### Checkpoint 1
Set up as much of the infrastructure as possible. At this point, the client, server, and payload should all be able to talk to each other in some capacity, even if it is done poorly. Due to the frankly ridiculuous volume of code required to achieve this, I expect the quality to suffer at the start.

### Checkpoint 2
I anticipate spending all of this time on cleanup, bug fixes, and making the code more idiomatic after the initial sprint to get everything working. I plan on spending this improving test case coverage, making thoughtful abstractions, and fixing as many bugs as I can discover.

### Final Submission
I plan on adding a very basic set of commands to the payload for controlling a compromised computer so that it has the functionality of a standard shell - things like listing directory contents, changing directory, reading, writing, encrypting and decrypting files. Additionally, I hope to implement **Malleable C2**, a common feature in C2 Frameworks that allows an operator to dynamically program the communication between the listener and the payload, but this is a nice to have at this point.

## Possible Challenges

I anticipate that the majority of the project is just simple engineering effort - things like setting up Postgres queries properly and getting some basic starting test coverage. However, I expect some real pain with the client, since it is a DLL that is not loaded from disk - that's not supposed to be possible! The OS will actively be fighting me, and I won't have the luxury of a debugger or any analysis tools - just good old fashioned logging to a file, since you can't even get a handle to stdout from memory. However, as I'm not doing anything too fancy, like custom CLR runtimes, self-encrypting programs, or custom WebAssembly runtimes, I don't expect any blocking problems.

## References
The backend code for the web server was based on Jeremy Chone's Rust10x series, although it has been modified HEAVILY. Regardless, not all of the work is my own, so I am going to link it here: https://github.com/rust10x/rust-web-app

If you're looking for an example of what a C2 framework actually is, I would recommend checking out the open source ones built by legitimate security companies, including Bishop Fox's Sliver (https://github.com/BishopFox/sliver) and BC-SECURITY's Powershell Empire (https://github.com/BC-SECURITY/Empire) 

