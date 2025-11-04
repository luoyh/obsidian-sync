```
2024年4月，腾讯游戏安全中心发布《无畏契约》禁止或不建议使用的软硬件列表，如果玩家使用可能会引起不必要的惩罚。

运行《无畏契约》时，禁止使用：

硬件：DMA、分线器/桥接器、KMbox、融合器、同步器、USB盒子、带宏脚本的鼠标等。

hack**工具：CheatEngine、YDArk、PowerTool、PCHunter、PYArk、APIMonitor、ProcMon、x96dbg、Windbg、OllydbgProcExplorer、ProcessHacker、ProcessMonitor、DbgPlugin、NoOne、Ke64、CrazyDbg等。

模拟类：按键精灵、具有恶意功能的OBS插件等。

虚拟化：VMWare、KVM、VT-X等。

运行《无畏契约》时，不建议使用：

远控和桌面共享软件：连连控、Todesk、向日葵、TeamViewer、Any-Desk、Steam Link、NvStreamer、Sun-shine、Parsec、moonlight等。

截帧工具：RenderDoc、WinPix、NVIDIA Nsight、Intel GPA等。

模拟类：AutoHotkey、DD键鼠驱动、大漠插件、python脚本等。

官方表示，游玩无畏契约的时候，使用以上软硬件，可能会出现弹框报错，或视情节严重程度，对账号进行不同时长的冻结或封禁处罚，比如30天、10年或永久禁止。

另外电脑做过机器码、使用鼠标宏、键盘管理软件、电脑里面有其它游戏作弊软件、系统U盘没拔、账号频繁异地登录或共享、共享别人的时候，别人开了按键精灵等程序、外接游戏手柄、队友作弊，自己被连坐等，都可能被禁止进入游戏，或账号被封禁处罚，会被反作弊检测到违规软件加载，游戏环境异常，修改游戏客户端违规等，而遭到封禁处罚。

腾讯游戏使用的是ACE反作弊，其它系列游戏大概率也是禁止使用以上软件或硬件的。远程软件等经常需要使用，可以关闭开机启动，只要避免远程软件与无畏契约等游戏同时运行，就没什么问题，当然有的软件虽然没有打开，但它却在后台运行，也可能导致被ACE反作弊误判违规。

```

### go语言实现的简单内存数值搜索,可绕过kk平台,使用管理员运行

```go
// ...existing code...
package main

import (
	"bufio"
	"encoding/binary"
	"fmt"
	"os"
	"strconv"
	"strings"
	"syscall"
	"unsafe"
)

// Windows API常量
const (
	PROCESS_ALL_ACCESS = 0x1F0FFF
	MEM_COMMIT         = 0x1000
	PAGE_READWRITE     = 0x04
	MAX_CANDIDATES     = 5 // 候选地址少于此时允许选择修改
)

// 内存区域信息结构
type MEMORY_BASIC_INFORMATION struct {
	BaseAddress       uintptr
	AllocationBase    uintptr
	AllocationProtect uint32
	RegionSize        uintptr
	State             uint32
	Protect           uint32
	Type              uint32
}

// 存储候选地址
var candidateAddrs []uintptr

// 存储上一次读取到的候选地址对应值（用于 incr/decr 筛选）
var prevValues = make(map[uintptr]uint32)

// 关注（watch）地址列表
var watchAddrs []uintptr

func main() {
	if len(os.Args) != 2 {
		fmt.Println("用法: go run main.go <进程PID>")
		return
	}

	help := `
se VALUE: 搜索VALUE的值（如果已有候选则在候选中筛选，否则全内存搜索）
re VALUE: 重新搜索新值，会清空候选并在全内存中搜索
incr: 筛选出比上次读取值增加的地址
decr: 筛选出比上次读取值减少的地址
ls: 打印当前候选值（只有候选值少于5个才打印详细地址和当前值，否则只显示数量）
ll: 打印当前关注（watch）地址及其当前值
add INDEX: 将候选列表中第INDEX个地址添加到关注列表（仅当候选数量小于等于5时有效）
set INDEX VALUE: 修改关注列表中第INDEX个地址的值（INDEX从1开始）
del INDEX: 删除关注列表中第INDEX个地址
clear: 清空候选、关注和历史值
exit: 退出
help: 显示帮助
（当候选地址少于等于5个时，会提示可直接选择一个地址进行修改，并可选择是否加入关注列表）
`

	// 解析PID
	pid, err := strconv.Atoi(os.Args[1])
	if err != nil {
		fmt.Printf("无效的PID: %v\n", err)
		return
	}

	// 加载Windows API
	kernel32 := syscall.NewLazyDLL("kernel32.dll")
	openProcess := kernel32.NewProc("OpenProcess")
	virtualQueryEx := kernel32.NewProc("VirtualQueryEx")
	readProcessMemory := kernel32.NewProc("ReadProcessMemory")
	writeProcessMemory := kernel32.NewProc("WriteProcessMemory")
	closeHandle := kernel32.NewProc("CloseHandle")

	// 打开进程
	handle, _, err := openProcess.Call(
		uintptr(PROCESS_ALL_ACCESS),
		0,
		uintptr(pid),
	)
	if handle == 0 || err != syscall.Errno(0) {
		fmt.Printf("打开进程失败: %v（请以管理员身份运行）\n", err)
		return
	}
	defer closeHandle.Call(handle)

	fmt.Println(help)
	reader := bufio.NewReader(os.Stdin)

	// 多轮命令循环
	for {
		fmt.Print("\ncmd> ")
		line, _ := reader.ReadString('\n')
		line = strings.TrimSpace(line)
		if line == "" {
			continue
		}
		parts := strings.Fields(line)
		cmd := strings.ToLower(parts[0])

		switch cmd {
		case "help":
			fmt.Println(help)

		case "exit":
			return

		case "clear":
			candidateAddrs = nil
			prevValues = make(map[uintptr]uint32)
			watchAddrs = nil
			fmt.Println("已清空候选、关注和历史值")

		case "se":
			if len(parts) != 2 {
				fmt.Println("用法: se VALUE")
				continue
			}
			target, err := strconv.Atoi(parts[1])
			if err != nil {
				fmt.Println("无效的数值")
				continue
			}
			if len(candidateAddrs) == 0 {
				fmt.Println("开始第一轮全内存搜索...（可能需要几分钟）")
				candidateAddrs = searchAllMemory(handle, virtualQueryEx, readProcessMemory, target)
			} else {
				fmt.Println("在候选地址中搜索精确匹配...")
				candidateAddrs = filterCandidates(handle, readProcessMemory, candidateAddrs, target)
			}
			afterSearchDisplay(handle, readProcessMemory, writeProcessMemory, reader)

		case "re":
			if len(parts) != 2 {
				fmt.Println("用法: re VALUE")
				continue
			}
			target, err := strconv.Atoi(parts[1])
			if err != nil {
				fmt.Println("无效的数值")
				continue
			}
			// 强制全量重新搜索
			prevValues = make(map[uintptr]uint32)
			fmt.Println("开始重新全内存搜索...（可能需要几分钟）")
			candidateAddrs = searchAllMemory(handle, virtualQueryEx, readProcessMemory, target)
			afterSearchDisplay(handle, readProcessMemory, writeProcessMemory, reader)

		case "incr":
			if len(candidateAddrs) == 0 {
				fmt.Println("当前没有候选地址，先用 se/re 搜索一个初始值")
				continue
			}
			candidateAddrs = filterIncr(handle, readProcessMemory, candidateAddrs)
			afterSearchDisplay(handle, readProcessMemory, writeProcessMemory, reader)

		case "decr":
			if len(candidateAddrs) == 0 {
				fmt.Println("当前没有候选地址，先用 se/re 搜索一个初始值")
				continue
			}
			candidateAddrs = filterDecr(handle, readProcessMemory, candidateAddrs)
			afterSearchDisplay(handle, readProcessMemory, writeProcessMemory, reader)

		case "ls":
			if len(candidateAddrs) == 0 {
				fmt.Println("当前没有候选地址")
			} else if len(candidateAddrs) <= MAX_CANDIDATES {
				fmt.Printf("找到%d个候选地址，以下是地址及当前值：\n", len(candidateAddrs))
				for i, addr := range candidateAddrs {
					val, ok := readMemoryValue(handle, readProcessMemory, addr)
					if ok {
						fmt.Printf("[%d] 地址: 0x%X  当前值: %d\n", i+1, addr, val)
					} else {
						fmt.Printf("[%d] 地址: 0x%X  当前值: 无法读取（可能已失效）\n", i+1, addr)
					}
				}
			} else {
				fmt.Printf("找到%d个候选地址，继续缩小范围（少于%d个时可查看详细信息）\n", len(candidateAddrs), MAX_CANDIDATES)
			}

		case "ll":
			if len(watchAddrs) == 0 {
				fmt.Println("关注列表为空")
			} else {
				fmt.Println("关注列表：")
				for i, addr := range watchAddrs {
					val, ok := readMemoryValue(handle, readProcessMemory, addr)
					if ok {
						fmt.Printf("[%d] 地址: 0x%X  当前值: %d\n", i+1, addr, val)
					} else {
						fmt.Printf("[%d] 地址: 0x%X  当前值: 无法读取（可能已失效）\n", i+1, addr)
					}
				}
			}

		case "add":
			// 仅允许在候选数量小于等于 MAX_CANDIDATES 时将候选项加入关注列表
			if len(candidateAddrs) == 0 {
				fmt.Println("当前没有候选地址，无法添加")
				continue
			}
			if len(candidateAddrs) > MAX_CANDIDATES {
				fmt.Printf("候选地址数量为 %d，超过 %d，无法直接添加到关注列表，请先缩小候选范围\n", len(candidateAddrs), MAX_CANDIDATES)
				continue
			}
			if len(parts) != 2 {
				fmt.Println("用法: add INDEX")
				continue
			}
			idx, err := strconv.Atoi(parts[1])
			if err != nil || idx <= 0 || idx > len(candidateAddrs) {
				fmt.Println("无效的索引")
				continue
			}
			addrToAdd := candidateAddrs[idx-1]
			already := false
			for _, a := range watchAddrs {
				if a == addrToAdd {
					already = true
					break
				}
			}
			if already {
				fmt.Printf("地址 0x%X 已在关注列表中\n", addrToAdd)
			} else {
				watchAddrs = append(watchAddrs, addrToAdd)
				fmt.Printf("已将候选地址 0x%X 添加到关注列表\n", addrToAdd)
			}

		case "set":
			if len(parts) != 3 {
				fmt.Println("用法: set INDEX VALUE")
				continue
			}
			idx, err1 := strconv.Atoi(parts[1])
			val, err2 := strconv.Atoi(parts[2])
			if err1 != nil || err2 != nil || idx <= 0 || idx > len(watchAddrs) {
				fmt.Println("无效的索引或数值")
				continue
			}
			targetAddr := watchAddrs[idx-1]
			ok := writeMemory(handle, writeProcessMemory, targetAddr, val)
			if ok {
				fmt.Printf("已将关注地址 0x%X 的值修改为 %d\n", targetAddr, val)
			} else {
				fmt.Println("修改失败")
			}

		case "del":
			if len(parts) != 2 {
				fmt.Println("用法: del INDEX")
				continue
			}
			idx, err := strconv.Atoi(parts[1])
			if err != nil || idx <= 0 || idx > len(watchAddrs) {
				fmt.Println("无效的索引")
				continue
			}
			removed := watchAddrs[idx-1]
			watchAddrs = append(watchAddrs[:idx-1], watchAddrs[idx:]...)
			fmt.Printf("已删除关注地址 0x%X\n", removed)

		default:
			fmt.Println("未知命令，输入 help 查看可用命令")
		}
	}
}

// 在每次搜索（或筛选）后统一显示并处理当候选数目较少时的交互
func afterSearchDisplay(handle uintptr, readProcessMemory, writeProcessMemory *syscall.LazyProc, reader *bufio.Reader) {
	if len(candidateAddrs) == 0 {
		fmt.Println("未找到匹配的地址，重置搜索")
		candidateAddrs = nil
		return
	}

	if len(candidateAddrs) <= MAX_CANDIDATES {
		fmt.Printf("\n找到%d个候选地址，以下是地址及当前值：\n", len(candidateAddrs))
		for i, addr := range candidateAddrs {
			val, ok := readMemoryValue(handle, readProcessMemory, addr)
			if ok {
				fmt.Printf("[%d] 地址: 0x%X  当前值: %d\n", i+1, addr, val)
				// 更新 prevValues 以便后续 incr/decr 使用
				prevValues[addr] = uint32(val)
			} else {
				fmt.Printf("[%d] 地址: 0x%X  当前值: 无法读取（可能已失效）\n", i+1, addr)
			}
		}

		// 选择要修改的地址
		fmt.Printf("\n请输入要修改的地址序号（1-%d，输入0取消）: ", len(candidateAddrs))
		choiceStr, _ := reader.ReadString('\n')
		choiceStr = strings.TrimSpace(choiceStr)
		choice, err := strconv.Atoi(choiceStr)
		if err != nil || choice < 0 || choice > len(candidateAddrs) {
			fmt.Println("无效选择，继续")
			return
		}
		if choice == 0 {
			fmt.Println("取消修改，继续")
			return
		}

		// 执行修改
		targetAddr := candidateAddrs[choice-1]
		fmt.Print("请输入新值: ")
		newValueStr, _ := reader.ReadString('\n')
		newValueStr = strings.TrimSpace(newValueStr)
		newValue, err := strconv.Atoi(newValueStr)
		if err != nil {
			fmt.Println("无效的新值，继续")
			return
		}
		success := writeMemory(handle, writeProcessMemory, targetAddr, newValue)
		if success {
			fmt.Printf("已将0x%X的值修改为%d\n", targetAddr, newValue)
			// 更新 prevValues
			prevValues[targetAddr] = uint32(newValue)
		} else {
			fmt.Println("修改失败")
		}

		// 是否加入关注列表
		fmt.Print("是否将该地址加入关注列表？(y/n): ")
		yn, _ := reader.ReadString('\n')
		yn = strings.TrimSpace(yn)
		if yn == "y" || yn == "Y" {
			already := false
			for _, a := range watchAddrs {
				if a == targetAddr {
					already = true
					break
				}
			}
			if !already {
				watchAddrs = append(watchAddrs, targetAddr)
				fmt.Printf("已将 0x%X 添加到关注列表\n", targetAddr)
			} else {
				fmt.Println("该地址已在关注列表中")
			}
		}
		return
	}

	fmt.Printf("找到%d个候选地址，继续搜索变化后的值（少于%d个时可选择修改）\n", len(candidateAddrs), MAX_CANDIDATES)
}

// 第一轮搜索：遍历所有内存区域，返回匹配目标值的地址列表
func searchAllMemory(handle uintptr, virtualQueryEx, readProcessMemory *syscall.LazyProc, target int) []uintptr {
	var addrs []uintptr
	var address uintptr = 0

	for {
		// 查询内存区域
		var mbi MEMORY_BASIC_INFORMATION
		mbiSize, _, err := virtualQueryEx.Call(
			handle,
			address,
			uintptr(unsafe.Pointer(&mbi)),
			uintptr(unsafe.Sizeof(mbi)),
		)
		if err != syscall.Errno(0) || mbiSize == 0 {
			break
		}

		// 筛选可读写的已提交内存
		if mbi.State == MEM_COMMIT && mbi.Protect&PAGE_READWRITE != 0 {
			// 读取内存数据
			// 注意：RegionSize 可能很大，可能导致内存使用高峰
			buf := make([]byte, mbi.RegionSize)
			var bytesRead uintptr
			success, _, err := readProcessMemory.Call(
				handle,
				mbi.BaseAddress,
				uintptr(unsafe.Pointer(&buf[0])),
				mbi.RegionSize,
				uintptr(unsafe.Pointer(&bytesRead)),
			)
			if success == 0 || err != syscall.Errno(0) || bytesRead == 0 {
				address = mbi.BaseAddress + mbi.RegionSize
				continue
			}

			// 查找4字节整数（小端序）
			for i := 0; i < len(buf)-4; i += 4 {
				val := binary.LittleEndian.Uint32(buf[i:])
				if uint32(target) == val {
					addr := mbi.BaseAddress + uintptr(i)
					addrs = append(addrs, addr)
					// 记录初始值
					prevValues[addr] = val
				}
			}
		}

		address = mbi.BaseAddress + mbi.RegionSize
	}

	return addrs
}

// 后续筛选：检查候选地址的当前值，保留匹配新目标的地址
func filterCandidates(handle uintptr, readProcessMemory *syscall.LazyProc, candidates []uintptr, target int) []uintptr {
	var newCandidates []uintptr
	var val uint32
	var bytesRead uintptr

	for _, addr := range candidates {
		// 读取该地址的4字节数据
		success, _, err := readProcessMemory.Call(
			handle,
			addr,
			uintptr(unsafe.Pointer(&val)),
			4,
			uintptr(unsafe.Pointer(&bytesRead)),
		)
		if success != 0 && err == syscall.Errno(0) && bytesRead == 4 && uint32(target) == val {
			newCandidates = append(newCandidates, addr)
			// 更新历史值
			prevValues[addr] = val
		}
	}

	return newCandidates
}

// 筛选出比上次读取值增加的地址
func filterIncr(handle uintptr, readProcessMemory *syscall.LazyProc, candidates []uintptr) []uintptr {
	var newCandidates []uintptr
	var val uint32
	var bytesRead uintptr

	for _, addr := range candidates {
		old, ok := prevValues[addr]
		if !ok {
			// 如果没有历史值则读取并记录但不保留（可以根据需要改变）
			success, _, err := readProcessMemory.Call(
				handle,
				addr,
				uintptr(unsafe.Pointer(&val)),
				4,
				uintptr(unsafe.Pointer(&bytesRead)),
			)
			if success != 0 && err == syscall.Errno(0) && bytesRead == 4 {
				prevValues[addr] = val
			}
			continue
		}
		success, _, err := readProcessMemory.Call(
			handle,
			addr,
			uintptr(unsafe.Pointer(&val)),
			4,
			uintptr(unsafe.Pointer(&bytesRead)),
		)
		if success != 0 && err == syscall.Errno(0) && bytesRead == 4 {
			if val > old {
				newCandidates = append(newCandidates, addr)
			}
			prevValues[addr] = val
		}
	}

	return newCandidates
}

// 筛选出比上次读取值减少的地址
func filterDecr(handle uintptr, readProcessMemory *syscall.LazyProc, candidates []uintptr) []uintptr {
	var newCandidates []uintptr
	var val uint32
	var bytesRead uintptr

	for _, addr := range candidates {
		old, ok := prevValues[addr]
		if !ok {
			success, _, err := readProcessMemory.Call(
				handle,
				addr,
				uintptr(unsafe.Pointer(&val)),
				4,
				uintptr(unsafe.Pointer(&bytesRead)),
			)
			if success != 0 && err == syscall.Errno(0) && bytesRead == 4 {
				prevValues[addr] = val
			}
			continue
		}
		success, _, err := readProcessMemory.Call(
			handle,
			addr,
			uintptr(unsafe.Pointer(&val)),
			4,
			uintptr(unsafe.Pointer(&bytesRead)),
		)
		if success != 0 && err == syscall.Errno(0) && bytesRead == 4 {
			if val < old {
				newCandidates = append(newCandidates, addr)
			}
			prevValues[addr] = val
		}
	}

	return newCandidates
}

// 读取指定地址的当前值（辅助函数，用于显示）
func readMemoryValue(handle uintptr, readProcessMemory *syscall.LazyProc, addr uintptr) (int, bool) {
	var val uint32
	var bytesRead uintptr
	success, _, err := readProcessMemory.Call(
		handle,
		addr,
		uintptr(unsafe.Pointer(&val)),
		4,
		uintptr(unsafe.Pointer(&bytesRead)),
	)
	if success != 0 && err == syscall.Errno(0) && bytesRead == 4 {
		return int(val), true
	}
	return 0, false
}

// 修改指定地址的内存值
func writeMemory(handle uintptr, writeProcessMemory *syscall.LazyProc, addr uintptr, value int) bool {
	var newVal uint32 = uint32(value)
	var bytesWritten uintptr

	success, _, err := writeProcessMemory.Call(
		handle,
		addr,
		uintptr(unsafe.Pointer(&newVal)),
		4,
		uintptr(unsafe.Pointer(&bytesWritten)),
	)

	return success != 0 && err == syscall.Errno(0) && bytesWritten == 4
}

```

### java版本

```java
package ce.t;

import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

import com.sun.jna.platform.win32.WinNT;

import ce.Kernel32;
import ce.MemoryModifier;
import ce.ProcessMemoryReader;

public class MemorySearchExample {
    private static final String help = """
            se VALUE: 搜索VALUE的值（如果已有候选则在候选中筛选，否则全内存搜索）
            re VALUE: 重新搜索新值，会清空候选并在全内存中搜索
            incr: 筛选出比上次读取值增加的地址
            decr: 筛选出比上次读取值减少的地址
            ls: 打印当前候选值（只有候选值少于5个才打印详细地址和当前值，否则只显示数量）
            ll: 打印当前关注（watch）地址及其当前值
            add INDEX: 将候选列表中第INDEX个地址添加到关注列表（仅当候选数量小于等于5时有效）
            set INDEX VALUE: 修改关注列表中第INDEX个地址的值（INDEX从1开始）
            del INDEX: 删除关注列表中第INDEX个地址
            clear: 清空候选、关注和历史值
            exit: 退出
            help: 显示帮助
            （当候选地址少于等于5个时，会提示可直接选择一个地址进行修改，并可选择是否加入关注列表）
            """;
    
    private static final int MAX_CANDIDATE = 5;
    public static void main(String[] args) {
        System.out.println(help);
        // 获取notepad进程ID（需要先启动notepad）

        int valueSize = 4;
        int processId = 4924;
        System.out.println("找到进程，PID: " + processId);

        // 打开进程
        WinNT.HANDLE hProcess = Kernel32.INSTANCE.OpenProcess(0x1F0FFF, false, processId);
        try {
            List<Long> candidateAddress = new ArrayList<>();
            List<Long> favoriteAddress = new ArrayList<>();
            boolean fx = false;
            System.out.print("cmd> ");
            try (Scanner scan = new Scanner(System.in)) {
                for (;;) {
                    if (fx) {
                        System.out.print("cmd> ");
                    }
                    fx = true;
                    String next = scan.nextLine();
                    int targetValue = 0;
                    int index = 0;
                    String[] parts = next.split("\\s+");
                    String cmd = parts[0];
                    switch (cmd) {
                    case "se":
                        if (parts.length < 2) {
                            System.out.println("输入错误,请输入要搜索的值, 如搜索1234:se 1234");
                            break;
                        }
                        targetValue = Integer.parseInt(parts[1]);
                        System.out.println("搜索整数值: " + targetValue);
                        if (candidateAddress.isEmpty()) {
                            MemorySearcher.searchIntegerInMemory(hProcess, targetValue, valueSize).forEach(candidateAddress::add);
                            System.out.println("\n找到 " + candidateAddress.size() + " 个匹配的地址:");
                            if (!candidateAddress.isEmpty() && candidateAddress.size() <= MAX_CANDIDATE) {
                                print(hProcess, candidateAddress, valueSize);
                                break;
                            }
                        } else {
                            List<Long> mdf = new ArrayList<>();
                            for (long address : candidateAddress) {
                                if (MemorySearcher.verifyAddress(hProcess, address, targetValue, valueSize)) {
                                    mdf.add(address);
                                }
                            }
                            candidateAddress.clear();
                            mdf.forEach(candidateAddress::add);
                            System.out.println("\n找到 " + candidateAddress.size() + " 个匹配的地址:");
                            if (!candidateAddress.isEmpty() && candidateAddress.size() <= MAX_CANDIDATE) {
                                print(hProcess, candidateAddress, valueSize);
                                break;
                            }
                        }
                        break;
                    case "re":
                        if (parts.length < 2) {
                            System.out.println("输入错误,请输入要搜索的值, 如搜索1234:re 1234");
                            break;
                        }
                        targetValue = Integer.parseInt(parts[1]);
                        System.out.println("搜索整数值: " + targetValue);
                        candidateAddress.clear();
                        
                        MemorySearcher.searchIntegerInMemory(hProcess, targetValue, valueSize).forEach(candidateAddress::add);
                        System.out.println("\n找到 " + candidateAddress.size() + " 个匹配的地址:");
                        if (!candidateAddress.isEmpty() && candidateAddress.size() <= MAX_CANDIDATE) {
                            print(hProcess, candidateAddress, valueSize);
                            break;
                        }
                        break;
                    case "ls":
                        System.out.println("找到 " + candidateAddress.size() + " 个匹配的地址:");
                        if (!candidateAddress.isEmpty() && candidateAddress.size() <= MAX_CANDIDATE) {
                            print(hProcess, candidateAddress, valueSize);
                        }
                        break;
                    case "ll":
                        System.out.println("关注 " + favoriteAddress.size() + " 个地址:");
                        if (!favoriteAddress.isEmpty()) {
                            print(hProcess, favoriteAddress, valueSize);
                        }
                        break;
                    case "add":
                        if (parts.length < 2) {
                            System.out.println("输入错误,请输入要添加的序号, 如:add 1");
                            break;
                        }
                        index = Integer.parseInt(parts[1]);
                        if (index <= 0 || index > candidateAddress.size()) {
                            System.out.println("当前候选数量为 " + candidateAddress.size() + " 请输入 1~" + candidateAddress.size() + " 之间的数字");
                            break;
                        }
                        if (candidateAddress.size() > 5) {
                            System.out.println("当前候选数量为 " +candidateAddress.size() +" 个,请缩小至5个以内添加");
                            break;
                        }
                        long addr = candidateAddress.get(index - 1);
                        System.out.println("添加地址: 0x%X -> %s%n".formatted(addr, MemorySearcher.addressValue(hProcess, addr, valueSize)));
                        boolean exist = false;
                        for (long fav : favoriteAddress) {
                            if (fav == addr) {
                                exist = true;
                                break;
                            }
                        }
                        if (!exist) favoriteAddress.add(addr);
                        System.out.println("添加关注列表成功,当前关注列表为:");
                        print(hProcess, favoriteAddress, valueSize);
                        break;
                    case "del":
                        if (parts.length < 2) {
                            System.out.println("输入错误,请输入要删除的序号, 如:del 1");
                            break;
                        }
                        index = Integer.parseInt(parts[1]);
                        if (index <= 0 || index > favoriteAddress.size()) {
                            System.out.println("当前关注数量为 " + favoriteAddress.size() + " 请输入 1~" + favoriteAddress.size() + " 之间的数字");
                            break;
                        }
                        favoriteAddress.remove(index  - 1);
                        System.out.println("删除关注成功,当前关注列表为:");
                        print(hProcess, favoriteAddress, valueSize);
                        break;
                    case "set":
                        if (parts.length < 3) {
                            System.out.println("输入错误,请输入要删除的序号, 如:set 1 321");
                            break;
                        }
                        index = Integer.parseInt(parts[1]);
                        targetValue = Integer.parseInt(parts[2]);
                        if (index <= 0 || index > favoriteAddress.size()) {
                            System.out.println("当前关注数量为 " + favoriteAddress.size() + " 请输入 1~" + favoriteAddress.size() + " 之间的数字");
                            break;
                        }
                        long ads = favoriteAddress.get(index - 1);
                        boolean success = MemoryModifier.modifyIntegerValue(hProcess, ads, targetValue, valueSize);
                        if (!success) {
                            System.out.println("修改内存失败: 0x%X -> %s".formatted(ads, MemorySearcher.addressValue(hProcess, ads, valueSize)));
                        } else {
                            System.out.println("修改成功, 当前关注列表值:");
                            print(hProcess, favoriteAddress, valueSize);
                        }
                        break;
                    case "help":
                        System.out.println(help);
                        break;
                    case "exit":
                        System.out.println("bye!");
                        return;
                        default: 
                            System.out.println("无法识别,输入help查看帮助");
                            break;
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            Kernel32.INSTANCE.CloseHandle(hProcess);
        }
    }
    
    private static void print(WinNT.HANDLE hProcess, List<Long> addresses, int valueSize) {
        int num = 1;
        for (long address : addresses) {
            System.out.printf("地址: [%d] 0x%X -> %s%n", num ++, address, MemorySearcher.addressValue(hProcess, address, valueSize));
        }
    }

    /**
     * 读取并显示指定地址周围的内存内容
     */
    private static void readAndDisplayMemory(WinNT.HANDLE hProcess, long address, int size) {
        try {
            // 读取地址前后的内存
            long readAddress = address - (size / 2);
            if (readAddress < 0)
                readAddress = 0;

            byte[] data = ProcessMemoryReader.readProcessMemory(hProcess, readAddress, size);

            System.out.printf("内存内容 (0x%X): ", readAddress);
            for (int i = 0; i < data.length; i++) {
                System.out.printf("%02X ", data[i]);
                if ((i + 1) % 8 == 0)
                    System.out.print(" ");
            }
            System.out.println();
//            
//            // 标记目标值的位置
//            int targetOffset = (int) (address - readAddress);
//            System.out.printf("目标位置: %s^%n", " ".repeat(targetOffset * 3));

        } catch (Exception e) {
            System.out.println("无法读取内存内容");
        }
    }
}


package ce.t;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import com.sun.jna.Pointer;
import com.sun.jna.platform.win32.WinNT;

import ce.ProcessMemoryReader;

public class MemorySearcher {
    
    /**
     * 在进程内存中搜索整数值
     * @param processId 进程ID
     * @param targetValue 要搜索的整数值
     * @param valueSize 值大小（4=32位，8=64位）
     * @return 找到的内存地址列表
     */
    public static List<Long> searchIntegerInMemory(WinNT.HANDLE hProcess, long targetValue, int valueSize) {
        List<Long> foundAddresses = new ArrayList<>();
        
        // 获取所有可读内存区域
        List<ProcessMemoryReader.MemoryRegion> regions = ProcessMemoryReader.getMemoryRegions(hProcess);
        
        System.out.println("开始搜索 " + regions.size() + " 个内存区域...");
        
        // 使用多线程加速搜索
        ExecutorService executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
        List<Future<List<Long>>> futures = new ArrayList<>();
        
        for (ProcessMemoryReader.MemoryRegion region : regions) {
            futures.add(executor.submit(() -> searchRegion(hProcess, region, targetValue, valueSize)));
        }
        
        // 收集结果
        for (Future<List<Long>> future : futures) {
            try {
                foundAddresses.addAll(future.get(10, TimeUnit.SECONDS));
            } catch (Exception e) {
                System.err.println("搜索区域时出错: " + e.getMessage());
            }
        }
        
        executor.shutdown();
        return foundAddresses;
    }
    
    /**
     * 在单个内存区域中搜索
     */
    public static List<Long> searchRegion(WinNT.HANDLE hProcess, ProcessMemoryReader.MemoryRegion region, 
                                         long targetValue, int valueSize) {
        List<Long> addresses = new ArrayList<>();
        long baseAddress = Pointer.nativeValue(region.baseAddress);
        long regionSize = region.size;
        
        // 跳过太小的区域
        if (regionSize < valueSize) {
            return addresses;
        }
        
        // 分批读取内存，避免一次性读取过大区域
        int chunkSize = 64 * 1024; // 64KB
        long chunks = (regionSize + chunkSize - 1) / chunkSize;
        
        for (long chunk = 0; chunk < chunks; chunk++) {
            long chunkStart = baseAddress + chunk * chunkSize;
            long chunkEnd = Math.min(baseAddress + regionSize, chunkStart + chunkSize);
            int readSize = (int) (chunkEnd - chunkStart);
            
            if (readSize < valueSize) continue;
            
            try {
                byte[] data = ProcessMemoryReader.readProcessMemory(hProcess, chunkStart, readSize);
                searchInBuffer(data, chunkStart, targetValue, valueSize, addresses);
                
            } catch (Exception e) {
                // 某些内存区域可能无法读取，忽略错误继续搜索其他区域
            }
        }
        
        return addresses;
    }
    
    /**
     * 在内存缓冲区中搜索目标值
     */
    private static void searchInBuffer(byte[] buffer, long baseAddress, 
                                     long targetValue, int valueSize, List<Long> addresses) {
        int maxOffset = buffer.length - valueSize;
        
        for (int offset = 0; offset <= maxOffset; offset += valueSize) {
            long value = bytesToLong(buffer, offset, valueSize);
            
            if (value == targetValue) {
                long foundAddress = baseAddress + offset;
                addresses.add(foundAddress);
            }
        }
    }
    
    /**
     * 将字节数组转换为long值（支持不同字节序）
     */
    private static long bytesToLong(byte[] bytes, int offset, int size) {
        long value = 0;
        
        // 小端序（x86/x64架构使用）
        for (int i = 0; i < size; i++) {
            value |= (bytes[offset + i] & 0xFFL) << (i * 8);
        }
        
        return value;
    }
    
    /**
     * 搜索字符串值
     */
    public static List<Long> searchStringInMemory(WinNT.HANDLE hProcess, String targetString) {
        List<Long> foundAddresses = new ArrayList<>();
        byte[] targetBytes = targetString.getBytes();
        
        List<ProcessMemoryReader.MemoryRegion> regions = 
            ProcessMemoryReader.getMemoryRegions(hProcess);
        
        for (ProcessMemoryReader.MemoryRegion region : regions) {
            long baseAddress = Pointer.nativeValue(region.baseAddress);
            long regionSize = region.size;
            
            if (regionSize < targetBytes.length) continue;
            
            int chunkSize = 64 * 1024;
            long chunks = (regionSize + chunkSize - 1) / chunkSize;
            
            for (long chunk = 0; chunk < chunks; chunk++) {
                long chunkStart = baseAddress + chunk * chunkSize;
                long chunkEnd = Math.min(baseAddress + regionSize, chunkStart + chunkSize);
                int readSize = (int) (chunkEnd - chunkStart);
                
                if (readSize < targetBytes.length) continue;
                
                try {
                    byte[] data = ProcessMemoryReader.readProcessMemory(hProcess, chunkStart, readSize);
                    searchBytesInBuffer(data, chunkStart, targetBytes, foundAddresses);
                    
                } catch (Exception e) {
                    // 忽略读取错误
                }
            }
        }
        
        return foundAddresses;
    }
    
    /**
     * 在缓冲区中搜索字节序列
     */
    private static void searchBytesInBuffer(byte[] buffer, long baseAddress, 
                                          byte[] targetBytes, List<Long> addresses) {
        int maxOffset = buffer.length - targetBytes.length;
        
        for (int offset = 0; offset <= maxOffset; offset++) {
            boolean found = true;
            
            for (int i = 0; i < targetBytes.length; i++) {
                if (buffer[offset + i] != targetBytes[i]) {
                    found = false;
                    break;
                }
            }
            
            if (found) {
                addresses.add(baseAddress + offset);
            }
        }
    }
    
    public static long addressValue(WinNT.HANDLE hProcess, long address, int valueSize) {
        byte[] data = ProcessMemoryReader.readProcessMemory(hProcess, address, valueSize);
        return bytesToLong(data, 0, valueSize);
    }
    
    /**
     * 验证找到的地址是否确实包含目标值
     */
    public static boolean verifyAddress(WinNT.HANDLE hProcess, long address, long targetValue, int valueSize) {
        try {
            byte[] data = ProcessMemoryReader.readProcessMemory(hProcess, address, valueSize);
            long actualValue = bytesToLong(data, 0, valueSize);
            return actualValue == targetValue;
        } catch (Exception e) {
            return false;
        }
    }
    
    /**
     * 监控内存值变化
     */
    public static void monitorMemoryValues(WinNT.HANDLE hProcess, List<Long> addresses, int valueSize) {
        System.out.println("开始监控 " + addresses.size() + " 个地址...");
        
        while (true) {
            System.out.println("============================");
            for (int i = 0; i < addresses.size(); i++) {
                long address = addresses.get(i);
                try {
                    byte[] data = ProcessMemoryReader.readProcessMemory(hProcess, address, valueSize);
                    long currentValue = bytesToLong(data, 0, valueSize);
                    System.out.printf("地址 0x%X: 值 = %d%n", address, currentValue);
                } catch (Exception e) {
                    System.out.printf("地址 0x%X: 无法读取%n", address);
                }
            }
            
            try {
                Thread.sleep(2000); // 每秒检查一次
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}


package ce;
import java.util.ArrayList;
import java.util.List;

import com.sun.jna.Memory;
import com.sun.jna.Pointer;
import com.sun.jna.platform.win32.WinNT;
import com.sun.jna.ptr.IntByReference;

public class ProcessMemoryReader {
    
    /**
     * 获取进程的内存区域信息
     */
    public static List<MemoryRegion> getMemoryRegions(WinNT.HANDLE hProcess) {
        List<MemoryRegion> regions = new ArrayList<>();
        
        try {
            Pointer address = new Pointer(0);
            Kernel32.MEMORY_BASIC_INFORMATION mbi = new Kernel32.MEMORY_BASIC_INFORMATION.ByReference();
            int mbiSize = mbi.size();
            for (;;) {
                // 遍历所有内存区域
                int result = Kernel32.INSTANCE.VirtualQueryEx(hProcess, address, mbi, mbiSize);
                if (result == 0) {
                    int lastError = Kernel32.INSTANCE.GetLastError();
                    System.out.println("last error: " + lastError);
                    break;
                }
                // 只处理已提交且可读的内存区域
                if (mbi.State == Kernel32.MEM_COMMIT && (mbi.Protect & Kernel32.PAGE_READWRITE) != 0) {
                    
                    MemoryRegion region = new MemoryRegion();
                    region.baseAddress = mbi.BaseAddress;
                    region.size = Pointer.nativeValue(mbi.RegionSize);
                    region.protect = mbi.Protect;
                    region.state = mbi.State;
                    
                    regions.add(region);
//                    int val = readMemory(hProcess, region.baseAddress, 4);
//                    System.out.println("valuex: " + val);
//                    if (val == 1235) {
//                        System.out.println("value: " + val);
//                    }
                }
                // 移动到下一个内存区域
                //address = mbi.BaseAddress.add(mbi.RegionSize.longValue());
                long nextAddr = Pointer.nativeValue(mbi.BaseAddress) + Pointer.nativeValue(mbi.RegionSize);
                address = new Pointer(nextAddr);
                // 检查是否到达内存末尾
                if (address.toString().equals("0x7fffffffffffffff") || 
                    address.toString().equals("0xffffffffffffffff")) {
                    break;
                }
            
            }
            
        } finally {
            //Kernel32.INSTANCE.CloseHandle(hProcess);
        }
        
        return regions;
    }
    

    public static int readMemory(WinNT.HANDLE process, Pointer address, int bytesToRead) {

        IntByReference read = new IntByReference(0);

        Memory output = new Memory(bytesToRead);

        Kernel32.INSTANCE.ReadProcessMemory(process, address, output, bytesToRead, read);

        return output.getInt(0);
    }
    
    /**
     * 读取进程内存数据
     */
    public static byte[] readProcessMemory(WinNT.HANDLE hProcess, long address, int size) {
        try {
            Memory buffer = new Memory(size);
            IntByReference bytesRead = new IntByReference();
            
            boolean success = Kernel32.INSTANCE.ReadProcessMemory(
                hProcess, new Pointer(address), buffer, size, bytesRead);
            
            if (success && bytesRead.getValue() == size) {
                return buffer.getByteArray(0, size);
            } else {
                throw new RuntimeException("读取内存失败，地址: " + Long.toHexString(address));
            }
            
        } finally {
            //Kernel32.INSTANCE.CloseHandle(hProcess);
        }
    }
    
    /**
     * 检查内存页是否可读
     */
    private static boolean isReadable(int protect) {
        return (protect & Kernel32.PAGE_READONLY) != 0 ||
               (protect & Kernel32.PAGE_READWRITE) != 0 ||
               (protect & Kernel32.PAGE_EXECUTE_READ) != 0 ||
               (protect & Kernel32.PAGE_EXECUTE_READWRITE) != 0;
    }
    
    /**
     * 内存区域信息类
     */
    public static class MemoryRegion {
        public Pointer baseAddress;
        public long size;
        public int protect;
        public int state;
        
        @Override
        public String toString() {
            return String.format("地址: %s, 大小: %d, 保护: 0x%X, 状态: 0x%X",
                baseAddress, size, protect, state);
        }
    }
}

package ce;

import com.sun.jna.Native;
import com.sun.jna.Pointer;
import com.sun.jna.Structure;
import com.sun.jna.platform.win32.WinDef;
import com.sun.jna.platform.win32.WinNT;
import com.sun.jna.ptr.IntByReference;
import com.sun.jna.win32.StdCallLibrary;

public interface Kernel32 extends StdCallLibrary {
    Kernel32 INSTANCE = Native.load("kernel32", Kernel32.class);

    // 内存状态常量
    int MEM_COMMIT = 0x1000;
    int MEM_RESERVE = 0x2000;
    int MEM_FREE = 0x10000;

    // 内存保护常量
    int PAGE_READONLY = 0x02;
    int PAGE_READWRITE = 0x04;
    int PAGE_EXECUTE_READ = 0x20;
    int PAGE_EXECUTE_READWRITE = 0x40;

    // VirtualQueryEx函数
    int VirtualQueryEx(WinNT.HANDLE hProcess, Pointer lpAddress, MEMORY_BASIC_INFORMATION lpBuffer, int dwLength);

    // ReadProcessMemory函数
    boolean ReadProcessMemory(WinNT.HANDLE hProcess, Pointer lpBaseAddress, Pointer lpBuffer, int nSize,
            IntByReference lpNumberOfBytesRead);

    // WriteProcessMemory函数
    boolean WriteProcessMemory(WinNT.HANDLE hProcess, Pointer lpBaseAddress,
                              Pointer lpBuffer, int nSize, IntByReference lpNumberOfBytesWritten);
 
    // OpenProcess函数
    WinNT.HANDLE OpenProcess(int dwDesiredAccess, boolean bInheritHandle, int dwProcessId);

    // VirtualProtectEx函数
    boolean VirtualProtectEx(WinNT.HANDLE hProcess, Pointer lpAddress,
                            int dwSize, int flNewProtect, 
                            IntByReference lpflOldProtect);
    // CloseHandle函数
    boolean CloseHandle(WinNT.HANDLE hObject);

    int GetLastError();


    // 进程访问权限
    int PROCESS_QUERY_INFORMATION = 0x0400;
    int PROCESS_VM_READ = 0x0010;

    // MEMORY_BASIC_INFORMATION结构体
    public static class MEMORY_BASIC_INFORMATION extends Structure {
        public static class ByReference extends MEMORY_BASIC_INFORMATION implements Structure.ByReference {}
        public Pointer BaseAddress;
        public Pointer AllocationBase;
        public int AllocationProtect;
        public Pointer RegionSize;
        public int State;
        public int Protect;
        public int Type;

        @Override
        protected java.util.List<String> getFieldOrder() {
            return java.util.Arrays.asList("BaseAddress", "AllocationBase", "AllocationProtect", "RegionSize", "State",
                    "Protect", "Type");
        }
    }
}


package ce;
import java.util.List;

import com.sun.jna.Memory;
import com.sun.jna.Pointer;
import com.sun.jna.platform.win32.WinNT;
import com.sun.jna.ptr.IntByReference;

public class MemoryModifier {
    
    /**
     * 修改进程内存中的整数值
     * @param processId 进程ID
     * @param address 内存地址
     * @param oldValue 旧值（用于验证）
     * @param newValue 新值
     * @param valueSize 值大小（4=32位，8=64位）
     * @return 是否修改成功
     */
    public static boolean modifyIntegerValue(WinNT.HANDLE hProcess, long address, long newValue, int valueSize) {
        try {
            // 第二步：修改内存权限（如果需要）
            if (!setMemoryProtection(hProcess, address, valueSize, Kernel32.PAGE_READWRITE)) {
                System.err.printf("无法修改内存权限：0x%X%n", address);
                return false;
            }
            
            // 第三步：写入新值
            if (!writeMemoryValue(hProcess, address, newValue, valueSize)) {
                System.err.printf("写入内存失败：0x%X%n", address);
                return false;
            }
            
//            // 第四步：验证新值
//            if (verifyCurrentValue(hProcess, address, newValue, valueSize)) {
//                System.out.printf("成功修改地址 0x%X: -> %d%n", address, newValue);
//                return true;
//            } else {
//                System.err.printf("修改后验证失败：0x%X%n", address);
//                return false;
//            }
            return true;
        } finally {
//            Kernel32.INSTANCE.CloseHandle(hProcess);
        }
    }
    
    /**
     * 验证当前内存值
     */
    private static boolean verifyCurrentValue(WinNT.HANDLE hProcess, long address, 
                                            long expectedValue, int valueSize) {
        try {
            Pointer buffer = new Memory(valueSize);
            IntByReference bytesRead = new IntByReference();
            
            boolean success = Kernel32.INSTANCE.ReadProcessMemory(
                hProcess, new Pointer(address), buffer, valueSize, bytesRead);
            
            if (success && bytesRead.getValue() == valueSize) {
                long currentValue = buffer.getLong(0);
                return currentValue == expectedValue;
            }
        } catch (Exception e) {
            System.err.println("验证值时出错: " + e.getMessage());
        }
        return false;
    }
    
    /**
     * 设置内存保护权限
     */
    private static boolean setMemoryProtection(WinNT.HANDLE hProcess, long address, 
                                             int size, int newProtect) {
        IntByReference oldProtect = new IntByReference();
        
        boolean success = Kernel32.INSTANCE.VirtualProtectEx(
            hProcess, new Pointer(address), size, 
            newProtect, oldProtect);
        
        if (success) {
            System.out.printf("内存权限修改成功：0x%X -> 0x%X%n", 
                oldProtect.getValue(), newProtect);
        }
        
        return success;
    }
    
    /**
     * 写入内存值
     */
    private static boolean writeMemoryValue(WinNT.HANDLE hProcess, long address, 
                                          long newValue, int valueSize) {
        Memory buffer = new Memory(valueSize);
        
        // 根据值大小写入不同的数据类型
        switch (valueSize) {
            case 1:
                buffer.setByte(0, (byte) newValue);
                break;
            case 2:
                buffer.setShort(0, (short) newValue);
                break;
            case 4:
                buffer.setInt(0, (int) newValue);
                break;
            case 8:
                buffer.setLong(0, newValue);
                break;
            default:
                throw new IllegalArgumentException("不支持的valueSize: " + valueSize);
        }
        
        IntByReference bytesWritten = new IntByReference();
        
        boolean success = Kernel32.INSTANCE.WriteProcessMemory(
            hProcess, new Pointer(address), buffer, valueSize, bytesWritten);
        
        return success && bytesWritten.getValue() == valueSize;
    }
    
    /**
     * 批量修改找到的所有1234值
     */
    public static int batchModifyValues(WinNT.HANDLE hProcess, List<Long> addresses, 
                                      long oldValue, long newValue, int valueSize) {
        int successCount = 0;
        int total = addresses.size();
        
        System.out.printf("开始批量修改 %d 个地址...%n", total);
        
        for (int i = 0; i < total; i++) {
            long address = addresses.get(i);
            System.out.printf("进度: %d/%d - ", i + 1, total);
            
            if (modifyIntegerValue(hProcess, address, newValue, valueSize)) {
                successCount++;
            }
            
            // 添加小延迟，避免过于频繁的操作
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                break;
            }
        }
        
        System.out.printf("批量修改完成：成功 %d/%d%n", successCount, total);
        return successCount;
    }
    
    /**
     * 修改字符串值
     */
    public static boolean modifyStringValue(WinNT.HANDLE hProcess, long address, 
                                          String oldValue, String newValue) {
        if (newValue.length() > oldValue.length()) {
            System.err.println("新字符串长度不能超过原字符串");
            return false;
        }
        
        try {
            // 验证当前字符串
            byte[] oldBytes = oldValue.getBytes();
            byte[] currentBytes = readMemoryBytes(hProcess, address, oldBytes.length);
            
            if (!java.util.Arrays.equals(currentBytes, oldBytes)) {
                System.err.println("字符串验证失败");
                return false;
            }
            
            // 修改内存权限
            if (!setMemoryProtection(hProcess, address, oldBytes.length, Kernel32.PAGE_READWRITE)) {
                return false;
            }
            
            // 写入新字符串
            byte[] newBytes = newValue.getBytes();
            byte[] writeBytes = new byte[oldBytes.length];
            System.arraycopy(newBytes, 0, writeBytes, 0, newBytes.length);
            // 剩余部分用0填充
            for (int i = newBytes.length; i < writeBytes.length; i++) {
                writeBytes[i] = 0;
            }
            
            return writeMemoryBytes(hProcess, address, writeBytes);
            
        } finally {
//            Kernel32.INSTANCE.CloseHandle(hProcess);
        }
    }
    
    /**
     * 读取内存字节
     */
    private static byte[] readMemoryBytes(WinNT.HANDLE hProcess, long address, int size) {
        Pointer buffer = new Memory(size);
        IntByReference bytesRead = new IntByReference();
        
        boolean success = Kernel32.INSTANCE.ReadProcessMemory(
            hProcess, new Pointer(address), buffer, size, bytesRead);
        
        if (success && bytesRead.getValue() == size) {
            return buffer.getByteArray(0, size);
        }
        throw new RuntimeException("读取内存失败");
    }
    
    /**
     * 写入内存字节
     */
    private static boolean writeMemoryBytes(WinNT.HANDLE hProcess, long address, byte[] data) {
        Memory buffer = new Memory(data.length);
        buffer.write(0, data, 0, data.length);
        
        IntByReference bytesWritten = new IntByReference();
        
        boolean success = Kernel32.INSTANCE.WriteProcessMemory(
            hProcess, new Pointer(address), buffer, data.length, bytesWritten);
        
        return success && bytesWritten.getValue() == data.length;
    }
}

        <dependency>
            <groupId>net.java.dev.jna</groupId>
            <artifactId>jna-platform</artifactId>
            <version>5.18.1</version>
        </dependency>
        <dependency>
            <groupId>net.java.dev.jna</groupId>
            <artifactId>jna</artifactId>
            <version>5.18.1</version>
        </dependency>

```