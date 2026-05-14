# IBA传输层

每个IBA数据包都包含一个传输头。传输报头包含端节点（endnode）完成指定操作所需的信息，例如，将数据有效载荷传递到端节点内的适当实体，例如线程或IO控制器。

IBA支持的传输服务类型

* 可靠连接（RC）
* 可靠数据报（RD）
* 扩展可靠连接（XRC）
* 不可靠数据报（UD）
* 不可靠连接（UC）

**非IBA协议封装服务包括**

* 原始IPv6数据报
* 原始以太网类型数据报

可靠传输服务使用序列号和确认消息（ACK/NAK）的组合来验证数据包的传递顺序，防止重复的数据包和无序的数据包被处理，并检测丢失的数据包。一旦检测到错误，例如丢失的数据包，请求者将重新传输丢失的数据包以及所有后续的数据包。**IBA不支持选择性分组重传，也不支持分组的无序接收**。

IBA操作被定义为包括请求消息和对于可靠服务的相应响应。因此，请求消息由请求者生成，而响应（如果存在）则由响应者生成。

![image-20230717141928686](image/IBA传输层/image-20230717141928686.png#pic_center)

请求消息由一个或多个IBA包组成。请求消息的包称为请求包。一个响应，除了RDMA读响应，只包含一个数据包。响应也叫acknoledge（确认）。表示一个或多个包被确认收到。响应可以确认接收包，所述包包括从请求消息的一部分到多个请求消息的任何位置的确认。

**不可靠的传输服务不使用确认消息。但是它们确实会生成序列号**。这允许响应程序检测无序或丢失的数据包，并执行本地恢复处理。不可靠数据报的任何恢复处理的细节都不在IBA规范的范围之内。





<table>
    <tr>
        <td colspan="2"><b><center>属性<center></b></td>
        <td><b><center>RC and XRC<center></b></td>
        <td><b><center>RD<center></b></td>
        <td><b><center>UD<center></b></td>
        <td><b><center>UC<center></b></td>
        <td><b>Raw数据报（IPv6和以太网类型）</b></td>
    </tr>
    <tr>
        <td colspan="2"><b>可扩展性</b>（N个处理器节点上的M个进程与所有节点上的所有进程通信）。</td>
        <td><b>RC</b>: 每个CA的每个处理器节点上需要<b>M<sup>2</sup>*N</b>个QP<br /><b>XRC</b>: 每个CA的每个处理器节点上需要<b>M*N</b>个QP。 </td>
        <td>每个CA，每个处理器节点上需要<b>M</b>个QP。</td>
        <td>每个CA，每个处理器节点上需要<b>M</b>个QP。</td>
        <td>每个CA，每个处理器节点上需要<b>M<sup>2</sup>*N</b>个QP。</td>
        <td>每个CA，每个端节点至少需要<b>1</b>个QP。</td>
    </tr>
    <tr>
        <td rowspan="5">可靠性</td>
        <td>检测到损坏的数据</td>
        <td colspan="5"><center><center>Yes</center></center></td>
	<tr>
    	<td>数据交付保证</td>
        <td colspan="2">数据只传递一次</td>
        <td colspan="3">没有保证</td>
    </tr>
    <tr>
        <td>保证数据顺序</td>
        <td>Yes，每个连接</td>
        <td>Yes，来自任何一个源QP的分组被排序到多个目的地QP。</td>
        <td><center>No</center></td>
        <td>检测到无序和重复的数据包。</td>
        <td><center>No</center></td>
    </tr>
    <tr>
    	<td>检测到数据丢失</td>
        <td colspan="2"><center>Yes</center></td>
        <td><center>No</center></td>
        <td><center>Yes</center></td>
        <td><center>No</center></td>
    </tr>
    <tr>
        <td>检测到数据丢失</td>
        <td colspan="2"><b>可靠性</b>：请求方和响应方都能检测到错误。请求方可以在没有客户端应用程序的任何参与下透明地从错误中恢复（重传、备用路径等）。只有在目标不可操作或通道适配器之间的所有结构路径均失败时，QP处理才会停止。</td>
        <td><b>不可靠</b>： 存在某些类型错误的数据包可能无法传送。 源 QP 和目标 QP 都不会收到丢弃数据包的通知</td>
        <td><b>不可靠</b>：响应者会检测并记录有错误（包括序列错误）的数据包。 请求者不会被告知。</td>
        <td><b>不可靠</b>：有错误的数据包不会被传送。 请求者和响应者不会收到丢弃数据包的通知。</td>
    </tr>
    <tr>
        <td>错误恢复</td>
    </tr>
</table>


## 基本传输头/BTH

基本传输标头（BTH）包含始终存在于所有IBA传输服务的字段——它不存在于原始数据包中。BTH的存在由Link Next Header指示(LRH:LNH)字段。

原始数据包不使用IBA传输服务，因此原始数据包不需要基本传输标头。

<center>所有IBA传输服务应包括一个基本传输头。</center>

![image-20230717151737131](image/IBA传输层/image-20230717151737131.png#pic_center)

