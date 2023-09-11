---
title: springboot查询数据库字段实践
category: Java
tag: 
 - Java
 - springboot

---

# springboot查询数据库字段实践

## 任务描述：

**在订单管理中增加购票信息界面、**

**查询条件：赛季名称、赛事名称、票种、姓名、身份证号、购买者、区域、座位、票类型**

**显示字段：赛季名称、赛事名称、票种、姓名、身份证号、购买者、区域、座位、二维码（限制显示长度20字符）、购票时间、票类型**

**可效仿订单管理页。**

## 后端实现步骤：

### 查看需要查询的表的字段

* 表名为`t_business_member_ticket`

* 字段分别为：
  * 赛季名称：通过activeId 查询t_business_activity 表的 season字段
  * 赛事名称：game_player字段
  * 票种：ticket_name字段
  * 姓名：bearer_name字段
  * 身份证号：bearer_id_num字段
  * 购买者：通过buyer字段的id 查询 t_business_member 表的name字段
  * 区域：通过 field_id 字段查询 t_base_venue_structure 表的data_name 字段
  * 座位：seat_name字段
  * 二维码：校验码（未定）write_off_code字段
  * 购票时间：buy_date 字段
  * 票类型：tickit_type 字段

```sql
SELECT t.id, t.ticket_name , t.bearer_id_num , t.bearer_name ,t.write_off_code,t.game_player , t.seat_name ,t.buy_date,t.tickit_type, m.name AS buyer_name , f.data_name AS field_name , a.activity_name AS activity_name 
FROM ticket.t_business_member_ticket t
JOIN t_business_member m ON t.buyer=m.id
JOIN t_base_venue_structure f ON t.field_id=f.id
JOIN t_business_activity a ON t.activity_id=a.id
```

### 后端采用的技术是 jpa

* 首先，创建需要返回的类 MemberTicketResponse.java

* 使用jpa进行多表联查。需要在repository文件中实现查询

  * ```java
    package com.jbrf.service.business.repository.custom;
    
    import com.jbrf.common.model.CustomBaseQuery;
    import com.jbrf.common.model.CustomPage;
    import com.jbrf.entity.basedata.QTBaseVenueStructure;
    import com.jbrf.entity.business.*;
    import com.jbrf.service.business.model.response.MemberTicketResponse;
    import com.querydsl.core.BooleanBuilder;
    import com.querydsl.core.types.Projections;
    import com.querydsl.jpa.impl.JPAQuery;
    import org.springframework.data.domain.Pageable;
    import org.springframework.stereotype.Repository;
    
    import java.util.List;
    
    import static org.apache.commons.lang3.StringUtils.isNotBlank;
    
    @Repository
    public class MemberTicketCustomRepository extends CustomBaseQuery<MemberTicket> {
        JPAQuery<MemberTicket> query = null;
        private QMemberTicket qMemberTicket = QMemberTicket.memberTicket;
        private QActivityInfo qActivityInfo = QActivityInfo.activityInfo;
        private QTBaseVenueStructure qtBaseVenueStructure = QTBaseVenueStructure.tBaseVenueStructure;
        private QMember qMember = QMember.member;
    
        public CustomPage<MemberTicketResponse> getMemberTicketByPage(
                Pageable pageable,
                String activityName,
                String gamePlayer,
                String ticketName,
                String bearerName,
                String bearer_id_num,
                String fieldName,
                String seatName,
                String ticketType,
                String buyerName
        ) {
            CustomPage<MemberTicketResponse> result = new CustomPage<MemberTicketResponse>(pageable);
    
            query = memberTicketPredict(activityName, gamePlayer, ticketName, bearerName, bearer_id_num,
                    fieldName, seatName, ticketType, buyerName);
    
            System.out.println(query);
            Long totalElements = query.select(qMemberTicket.id).fetchCount();
            if (null == totalElements || totalElements == 0L) {
                return result;
            }
            List<MemberTicketResponse> ticketResponseList = query
                    .orderBy(qMemberTicket.buyDate.desc())
                    .offset(pageable.getOffset())
                    .limit(pageable.getPageSize())
                    .select(
                            Projections.constructor(MemberTicketResponse.class, qMemberTicket.id, qMemberTicket.buyDate, qMemberTicket.seatName, qMember.name, qMemberTicket.ticketType, qMemberTicket.bearerName, qMemberTicket.bearerIdNum, qMemberTicket.gamePlayer, qMemberTicket.ticketName, qtBaseVenueStructure.dataName, qActivityInfo.activityName, qMemberTicket.writeOffCode)
                    )
                    .fetch();
    
            return result.totalElements(totalElements).content(ticketResponseList);
        }
    
    
        private JPAQuery<MemberTicket> memberTicketPredict(String activityName,
                                                           String gamePlayer,
                                                           String ticketName,
                                                           String bearerName,
                                                           String bearerIdNum,
                                                           String fieldName,
                                                           String seatName,
                                                           String ticketType,
                                                           String buyerName) {
            BooleanBuilder predicate = new BooleanBuilder();
    
            if (isNotBlank(activityName)) {
                predicate.and(qActivityInfo.activityName.contains(activityName.trim()));
            }
            if (isNotBlank(gamePlayer)) {
                predicate.and(qMemberTicket.gamePlayer.contains(gamePlayer.trim()));
            }
            if (isNotBlank(ticketName)) {
                predicate.and(qMemberTicket.ticketName.contains(ticketName.trim()));
            }
            if (isNotBlank(bearerName)) {
                predicate.and(qMemberTicket.bearerName.contains(bearerName.trim()));
            }
            if (isNotBlank(bearerIdNum)) {
                predicate.and(qMemberTicket.bearerIdNum.eq(bearerIdNum.trim()));
            }
            if (isNotBlank(fieldName)) {
                predicate.and(qtBaseVenueStructure.dataName.contains(fieldName.trim()));
            }
            if (isNotBlank(seatName)) {
                predicate.and(qMemberTicket.seatName.contains(seatName.trim()));
            }
            if (isNotBlank(ticketType)) {
                predicate.and(qMemberTicket.ticketType.in(ticketType.split(",")));
            }
            if (isNotBlank(buyerName)) {
                predicate.and(qMember.name.eq(buyerName.trim()));
            }
            return query().from(qMemberTicket)
                    .leftJoin(qMember).on(qMemberTicket.buyer.eq(qMember.id))              .leftJoin(qtBaseVenueStructure).on(qMemberTicket.fieldId.eq(qtBaseVenueStructure.id))
                    .leftJoin(qActivityInfo).on(qMemberTicket.activityId.eq(qActivityInfo.id))
                    .where(predicate);
        }
    }
    
    ```

  * 这段代码动态创建查询语句，并且将查询结果封装给要返回的类 MemberTicketResponse

### 在service 和 serviceImpl中定义，最后在controller中实现

## 前端实现步骤

前端对api进行了封装，封装到api下的js中

```js
import request from '@/utils/request'

const MemberTicket = {
  // 分页查询会员票列表
  memberTicketList(params) {
    return request({
      url: `http://localhost:51066/memberTicket/page`,
      method: 'get',
      params
    })
  }
}
export default MemberTicket

 //引用
import MemeberTicket from ./memberTicket.js
MemberTicker.memberTicketList(data).then(res=>{})

//其中request是对axios的封装
```

