## 이전 시간 접속 문제

처음에는 operation timed out, permission denied가 나왔었는데

나중에 재접속 했을 때에 위처럼 `remote host key has changed, port forwarding is disabled` 떴음

- 키 정보 변경된 것이 반영이 안 되어서 재설정함

```jsx
ssh-keygen -R jiwon-homelab
```

하고 재접속 시도

- 근데 아직 디바이스 ACL 쪽에서 권한이 잘 부여되지 않았음
- 최종 ACL
    
    ```jsx
    // Example/default ACLs for unrestricted connections.
    {
    	// Declare static groups of users. Use autogroups for all users or users with a specific role.
    	"groups": {
    		"group:admin": ["kooriangman@gmail.com", "sarahlee0110@gmail.com"],
    	},
    
    	// Define the tags which can be applied to devices and by which users.
    	"tagOwners": {
    		"tag:admin":    ["kooriangman@gmail.com"],
    		"tag:home-lab": ["kooriangman@gmail.com"],
    	},
    
    	// Define access control lists for users, groups, autogroups, tags,
    	// Tailscale IP addresses, and subnet ranges.
    	"acls": [
    		// Allow all connections.
    		// Comment this section out if you want to define specific restrictions.
    		//{"action": "accept", "src": ["group:admin"], "dst": ["tag:home-lab:*"]},
    
    		{"action": "accept", "src": ["group:admin"], "dst": ["tag:home-lab:*"]},
    
    		// Allow users in "group:example" to access "tag:example", but only from
    		// devices that are running macOS and have enabled Tailscale client auto-updating.
    		// {"action": "accept", "src": ["group:example"], "dst": ["tag:example:*"], "srcPosture":["posture:autoUpdateMac"]},
    	],
    
    	// Define postures that will be applied to all rules without any specific
    	// srcPosture definition.
    	// "defaultSrcPosture": [
    	//      "posture:anyMac",
    	// ],
    
    	// Define device posture rules requiring devices to meet
    	// certain criteria to access parts of your system.
    	// "postures": {
    	//      // Require devices running macOS, a stable Tailscale
    	//      // version and auto update enabled for Tailscale.
    	// 	"posture:autoUpdateMac": [
    	// 	    "node:os == 'macos'",
    	// 	    "node:tsReleaseTrack == 'stable'",
    	// 	    "node:tsAutoUpdate",
    	// 	],
    	//      // Require devices running macOS and a stable
    	//      // Tailscale version.
    	// 	"posture:anyMac": [
    	// 	    "node:os == 'macos'",
    	// 	    "node:tsReleaseTrack == 'stable'",
    	// 	],
    	// },
    
    	// Define users and devices that can use Tailscale SSH.
    	"ssh": [
    		// Allow all users to SSH into their own devices in check mode.
    		// Comment this section out if you want to define specific restrictions.
    		{
    			"action": "check",
    			"src":    ["autogroup:member"],
    			"dst":    ["autogroup:self"],
    			"users":  ["autogroup:nonroot", "root"],
    		},
    		{
    			"action":      "check",
    			"src":         ["group:admin"],
    			"dst":         ["tag:home-lab"],
    			"users":       ["autogroup:nonroot", "root"],
    			"checkPeriod": "3h",
    		},
    	],
    
    	// Test access rules every time they're saved.
    	// "tests": [
    	//  	{
    	//  		"src": "alice@example.com",
    	//  		"accept": ["tag:example"],
    	//  		"deny": ["100.101.102.103:443"],
    	//  	},
    	// ],
    }
    
    ```
    

원인 : group:admin의 ssh 접근을 허용하지 않았어서 주석 해제해주니 접속 잘 됨

## 서버 태그 붙여서 연결

```bash
sudo tailscale up --advertise-exit-node --ssh --advertise-tags=tag:home-lab
```

## crontab 설정

서버가 상태가 좋지 않아서.. 일단은 매일 0시 0분에 리부팅 되도록 세팅했다

```bash
# 전체 cron 파일 설정
vi /etc/crontab
# 정각에 너무 칼 같이 하면 작업하다가 까먹을까봐.. 5분 유예 줌. 그리 의미 있는 지는 잘?
0 0 * * * root /bin/sleep 300 && /sbin/shutdown -r now
```