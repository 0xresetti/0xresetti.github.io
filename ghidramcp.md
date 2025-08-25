# Setting up & Configuring GhidraMCP - Using AI for quick Malware Analysis

### A while back, I came across an awesome video by [LaurieWired](https://www.youtube.com/watch?v=u2vQapLAW88) which was talking about GhidraMCP, which is essentially an MCP (Model Context Protocol)designed for use with Ghidra. In this blog I will walk through the setup and use of this program, as I believe It's capabilities are highly useful for an initial overview or quick analysis of a malicious file.

### Prerequisites 

First of all, go check out the [Github repo](https://github.com/LaurieWired/GhidraMCP) for this project, it's ideal to read alongside this post. 

Then, download the latest [GhidraMCP release,](https://github.com/LaurieWired/GhidraMCP/releases/download/1.4/GhidraMCP-release-1-4.zip) and extract it to a folder of your choice, you should have two files, `"GhidraMCP-1-4.zip"` (do NOT unzip the .zip) and `"bridge_mcp_ghidra.py"`, as seen below:

<img width="422" height="246" alt="image" src="https://github.com/user-attachments/assets/1ea45b41-fecf-4b7c-ad30-960b9016a6e3" />

After that, install the latest version of [Python](https://www.python.org/ftp/python/3.13.0/python-3.13.0-amd64.exe). If you want to go with the exact same version I use, I use 3.13.

Make sure to check "Add Python 3.13 to PATH" when installing.

<img width="832" height="508" alt="image" src="https://github.com/user-attachments/assets/bf9558fb-a507-4f84-a49f-12729bdb506e" />

After that, you should be able to go into a CMD prompt and run `python --version` and see that python has been successfully installed.

<img width="408" height="105" alt="image" src="https://github.com/user-attachments/assets/d1eb7b56-e9b7-4d0a-aa6e-8e30d56b0928" />

Since `pip` comes preinstalled with Python, you should be able to run the next required command, which will install the [MCP SDK](https://github.com/modelcontextprotocol/python-sdk) which is required for this project to work fully, the command is: `pip install "mcp[cli]"`

<img width="1460" height="382" alt="image" src="https://github.com/user-attachments/assets/6c90d8e0-4ad4-4f2b-8350-7d90b36cf028" />

(Be sure to update your pip guys !!1!11!!!!1!)

Following that, you should install the [Claude Desktop](https://claude.ai/download) app, you can also use some different AI agents for this, some examples from Laurie's repo include [Cline](https://cline.bot/), and [5ire](https://github.com/nanbingxyz/5ire)

Finally, if you don't already have Ghidra installed and setup, then I don't even know why you are reading this blog post, however it should be made known that GhidraMCP only supports versions of Ghidra up to version 11.3.2, which you can download from [this link.](https://github.com/NationalSecurityAgency/ghidra/releases/download/Ghidra_11.3.2_build/ghidra_11.3.2_PUBLIC_20250415.zip) Hopefully Laurie will update the extension to the latest versions of Ghidra.

After installing Claude and that specific version of Ghidra, you should have everything downloaded and installed and be ready to set it all up.

### Installation

Open up Ghidra, and go to `File` > `Install Extensions`, you should see the box below

<img width="887" height="836" alt="image" src="https://github.com/user-attachments/assets/79a25c7a-1ac3-43ab-b8ab-669ee48673a1" />

Select the big green plus sign in the top right and browse to where you saved the `"GhidraMCP-1-4.zip"` zip file, select it, and click OK, you should now have the GhidraMCP extension installed, make sure that the checkbox next to it is checked:

<img width="813" height="139" alt="image" src="https://github.com/user-attachments/assets/5762f73b-0d5d-4345-8858-d9be397e059f" />

Then restart Ghidra for extensions changes to take effect.

When you have restarted Ghidra, put a random binary into it, don't bother doing the initial auto-analysis, just click No, then go to `File` > `Configure` > (Make sure `Developer` is checked) then click `Configure` underneath `Developer` > Make sure `GhidraMCPPlugin` is checked, then click OK.

<img width="387" height="803" alt="image" src="https://github.com/user-attachments/assets/8489fbe4-6d0f-4ac0-b212-ba66740d8e3b" />

<img width="1097" height="830" alt="image" src="https://github.com/user-attachments/assets/f3750e8f-2d23-4a31-b35b-21241fc4a3ac" />

<img width="882" height="339" alt="image" src="https://github.com/user-attachments/assets/6e483727-7a4e-4f9c-9d04-896bbb0b9c88" />


After that, you can open Claude Desktop, and go to `Claude` > `Settings` > `Developer` > `Edit Config` > `claude_desktop_config.json` and add the following json data into it:

```json
{
  "mcpServers": {
    "ghidra": {
      "command": "python",
      "args": [
        "\\ABSOLUTE_PATH_TO\\bridge_mcp_ghidra.py",
        "--ghidra-server",
        "http://127.0.0.1:8080/"
      ]
    }
  }
}
```

For example, since my GhidraMCP folder and `bridge_mcp_ghidra.py` is on my Desktop, my config will look like this:

```json
{
  "mcpServers": {
    "ghidra": {
      "command": "python",
      "args": [
        "C:\\Users\\resetti\\Desktop\\GhidraMCP\\bridge_mcp_ghidra.py",
        "--ghidra-server",
        "http://127.0.0.1:8080/"
      ]
    }
  }
}
```

You should use my config as a guide for what your config should look like.

Once you have edited that config, save it, and close out Claude completely, this includes closing Claude from Task Manager or the little up arrow task bar thing on Windows I have no idea what its called lol.

Then, you should re
