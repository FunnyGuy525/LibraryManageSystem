# LibraryManageSystem
简单的图书管理系统(mysql)
​

## 数据库结构设计

### E-R图

### 数据库创建语句

create database if not exists tsjy;

use tsjy;

### 表结构设计说明

借阅人表是一个实体表，它存储了借阅人的基本信息，如证件号、姓名、类别、已借数目和电话。证件号是借阅人表的主键，它可以唯一标识每个借阅人。类别字段有一个检查约束，它限制了类别只能是教师或学生。已借数目字段有一个大于等于0的约束表示借书数量不会为负数，以及一个默认值0，它表示每个借阅人初始时没有借阅任何图书。

图书表是另一个实体表，它存储了图书的基本信息，如图书编号、书名、类别和是否借出。图书编号是图书表的主键，它可以唯一标识每本图书。是否借出字段是一个布尔类型，它表示图书是否已经被借出。类别字段有一个检查约束，它限制了每本书只能是已有的某一种类别。是否借出字段有一个默认值，它表示每本图书初始时都没有被借出。

借阅信息表是一个关系表，它存储了借阅人和图书之间的多对多关系，以及相关的借阅信息，如借出日期、应归还日期和实际归还日期。借阅信息表有两个外键：证件号和图书编号，它们分别引用了借阅人表和图书表的主键。这样就可以确保每条借阅信息都对应一个有效的借阅人和一个有效的图书。借出日期字段是一个日期/时间类型，它表示图书被借出的日期。应归还日期字段也是一个日期/时间类型，它表示图书应该归还的日期。应归还日期字段是一个计算字段，它根据借出日期加上30天来计算。实际归还日期字段也是一个日期/时间类型，它表示图书实际归还的日期。

### 数据表创建语句

create table borrowers(

     id char(6) not null,

    name char(10) not null,

    type enum('teacher', 'student') NOT NULL,

    amount int not null default 0 check(amount >= 0),

    phone char(11) not null,

     primary key(id)

);

create table books(

     bookid char(20) not null,

    bookname char(20) not null,

    category enum('文学', '理学', '法学', '工学', '其他') not null,

    in_out boolean not null default false,

    primary key(bookid)

);

create table borrow_info(

     borrow_id int not null auto_increment,

    id char(6) not null,

    bookid char(20) not null,

    borrow_date date not null,

    due_date date generated always as (date_add(borrow_date, interval 30 day)),

    return_date date ,

    primary key(borrow_id),

    foreign key(id)references borrowers(id),

    foreign key(bookid)references books(bookid),

    check(return_date >= borrow_date or return_date is null)

);

## 功能设置

### 数据操作测试。

1.创建视图显示所有逾期未归还的借阅信息（包括借阅人姓名，借阅人类别，书名，借出日期，应归还日期，逾期时长）

create view overdue_info(借阅人姓名,借阅人类别,书名,借出日期,应归还日期,逾期时长) as

select name,type,bookname,borrow_date,due_date,datediff(curdate(),due_date)

     from borrow_info bi join borrowers b on bi.id = b.id

    join books bk on bi.bookid = bk.bookid

    where bi.return_date is null and curdate() > bi.due_date;

2.存储过程首先应当判断书籍未被借出去时借阅才能成功，

#数据才可以正确插入，否则借阅失败，同时抛出一个自定义的错误信息，提示这本书已经被借出。

#借阅人表中的借阅数量的修改可以通过触发器实现

delimiter $$

create procedure borrow_book(in p_id char(6),in p_bookid char(20))

begin

  declare v_in_out boolean;

  select in_out into v_in_out from books where bookid = p_bookid;

  if v_in_out = false then

    # update borrowers set amount = amount + 1 where id = p_id;

    update books set in_out = true where bookid = p_bookid;

    insert into borrow_info(id, bookid, borrow_date) values(p_id, p_bookid, curdate());

  else

    signal sqlstate '45000' set message_text = '书籍已被借出';

  end if;

end$$

delimiter ;

3. 还书将书籍在库信息修改为true，表示归还入库，同时将今天的日期填入借阅信息表中的归还日期。

#借阅人表中的借阅数量的修改可以通过触发器实现

delimiter $$

create procedure return_book(in p_id char(6),in p_bookid char(20))

begin

     # update borrowers set amount = amount - 1 where id = p_id;

    update books set in_out = false where bookid = p_id;

    update borrow_info set return_date = curdate() where id = p_id and bookid = p_bookid;

end$$

delimiter ;

4.查询借阅人姓名

delimiter $$

create function get_borrower_name(p_bookid char(20)) returns char(10)

begin

     declare v_name char(10);

    select name into v_name from borrowers b

    join borrow_info bi on b.id = bi.id

    join books bk on bk.bookid = bi.bookid

    where bi.bookid = p_bookid;

    return v_name;

end$$

delimiter ;

5.计算某借阅人还能借阅的图书数目，学生限额5本，教师限额10本

delimiter $$

create function get_available_books(p_id char(6)) returns int

begin

     declare v_type enum('teacher', 'student');

    declare v_amount int;

    select type,amount into v_type,v_amount from borrowers where id = p_id;

    if v_type = 'teacher' then

           return (10 - v_amount);

     else

           return (5 - v_amount);

     end if;

end$$

delimiter ;

6.查询某本图书逾期未还的时长

delimiter $$

create function get_overdue_days(p_bookid char(20)) returns int

begin

     declare v_days int;

    select datediff(curdate(),due_date) into v_days

       from borrow_info where bookid = p_bookid and return_date is null;

     if v_days > 0 then

           return v_days;

     else

           return 0;

     end if;

end$$

delimiter ;

#调用该函数显示所有逾期未归还图书的书名，

#借阅人和逾期时长并按逾期时长排序

select bk.bookname,b.name,get_overdue_days(bk.bookid) as 逾期时长

from borrow_info bi join books bk on bk.bookid = bi.bookid

join borrowers b on b.id = bi.id

where get_overdue_days(bk.bookid) > 0

order by 逾期时长 desc;

#7.查询某借阅人有几本逾期未还图书

delimiter $$

create function get_overdue_books(p_id char(6)) returns int

begin

     declare count int default 0;

    select count(*) into count from borrow_info where id = p_id

           and return_date is null and curdate() > due_date;

     return count;

end$$

delimiter ;

#调用该函数显示有逾期未归还图书的借阅人和未归还图书数目

select name, get_overdue_books(id) as overdue_books

     from borrowers where get_overdue_books(id) > 0;

8.利用游标计算计算某借阅人逾期未还图书应缴纳的罚款，逾期30日内罚款1元，逾期90日内罚款3元，逾期超过90日罚款5元。

delimiter $$

create function get_fine(p_id char(6)) returns decimal(10,2)

begin

     declare v_bookid char(20);

    declare v_borrow_date date;

    declare v_due_date date;

    declare v_overdue_days int;

    declare v_fine decimal(10,2) default 0;

    declare done int default false;

    declare cur cursor for select bookid, borrow_date, due_date from borrow_info

    where id = p_id and return_date is null and due_date < curdate();

    declare continue handler for not found set done = true;

   

    open cur;

    loop1: loop

        fetch cur into v_bookid, v_borrow_date, v_due_date;

        if done then

          leave loop1;

        end if;

        set v_overdue_days = datediff(curdate(), v_due_date);

        if v_overdue_days <= 30 then

          set v_fine = v_fine + 1;

        elseif v_overdue_days <= 90 then

          set v_fine = v_fine + 3;

        else

          set v_fine = v_fine + 5;

        end if;

    end loop loop1;

    close cur;

   

    return v_fine;

end$$

delimiter ;

#调用该函数显示所有应缴纳罚款的借阅人的姓名，逾期罚款和电话

select name, get_fine(id) as fine, phone from borrowers

where get_fine(id) > 0;

#9.创建两个触发器，分别在借出或归还图书时，修改借阅人表中的已借数目字段

delimiter $$

create trigger borrow_book after insert on borrow_info for each row

begin

     update borrowers set amount = amount + 1 where id = new.id;

end$$

delimiter ;

delimiter $$

create trigger return_book after update on borrow_info for each row

begin

     update borrowers set amount = amount - 1 where id = new.id;

end$$

delimiter ;

10.创建触发器，当借阅者已借阅的书籍数目达到限额时，禁止借入新的书籍

delimiter $$

create trigger t before insert on borrow_info for each row

begin

     declare v_amount int;

    declare v_type enum('teacher', 'student');

    select type, amount into v_type, v_amount from borrowers where id = new.id;

    if v_type = 'teacher' and v_amount >= 10 then

        signal sqlstate '45000' set message_text = 'Teacher cannot borrow more than 10 books';

    elseif v_type = 'student' and v_amount >= 5 then

        signal sqlstate '45000' set message_text = 'Student cannot borrow more than 5 books';

    end if;

end$$

delimiter ;

增加功能

11. 创建存储函数，查询某本图书的借阅次数，并调用该函数显示最受欢迎的5本图书；

delimiter $$

create function book_borrow_count(p_bookid char(20)) returns int

begin

     declare count int;

     select count(*) into count from borrow_info where bookid = p_bookid;

end$$

delimiter ;

select bookid, bookname, book_borrow_count(bookid) as borrow_count

from books order by borrow_count desc limit 5;

12. 创建存储函数，查询某借阅人最喜欢的图书类别；

delimiter $$

create function favoritecategory (id char(6))

returns varchar(10)

deterministic

begin

  declare fav_category varchar(10);

  declare count int;

  select category, count(*)

  into fav_category, count

  from borrow_info join books on borrow_info.bookid = books.bookid

  where id = borrow_info.id

  group by category

  order by count desc

  limit 1;

  return fav_category;

end $$

delimiter ;

13. 创建存储过程，每周定期生成一份借阅报告，包括借阅人数，图书数，类别分布，逾期情况并显示。

delimiter $$

create procedure generate_report()

begin

     declare v_borrower_count int;

    declare v_book_count int;

     declare v_overdue_count int;

    declare v_overdue_rate decimal(5,2);

     select count(distinct id) into v_borrower_count from borrow_info;

     select count(distinct bookid) into v_book_count from borrow_info;

     select count(*) into v_overdue_count from borrow_info where return_date is null and due_date < curdate();

     set v_overdue_rate = v_overdue_count / v_book_count * 100;

     select concat(‘本周借阅报告：’) as report;

     select concat(‘借阅人数：’, v_borrower_count) as borrower_count;

     select concat(‘借阅图书数：’, v_book_count) as book_count;

     select concat(‘逾期数：’, v_overdue_count) as overdue_count;

     select concat(‘逾期率：’, v_overdue_rate, ‘%’) as overdue_rate;

    select concat(‘类别分布：’) as category_distribution;

     select category, count(distinct bookid) as book_count from borrow_info join books on borrow_info.bookid = books.bookid group by category;

end;

delimiter ;

#创建事件调度器，每周定时执行存储过程（假设每周一早上9点）

create event generate_report_event on schedule every 1 week

starts '2023-05-22 09:00:00'

do call generate_report();

## 测试数据库对象及其关联

-- 插入一些借阅者数据

insert into borrowers values

('100001', '张三', 'teacher', 2, '13812345678'),

('100002', '李四', 'student', 0, '13987654321'),

('100003', '王五', 'student', 1, '13765432198');

-- 这些数据能正确插入，因为它们满足了表的约束条件

-- 尝试插入一个不存在的类型

insert into borrowers values

('100004', '赵六', 'staff', 0, '13612349876');

-- 这条数据不能插入，因为type列只能是teacher或student

-- 尝试插入一个负数的借书数量

insert into borrowers values

('100005', '孙七', 'student', -1, '13598761234');

-- 这条数据不能插入，因为amount列有一个check约束，要求amount >= 0

-- 修改一个借阅者的电话号码

update borrowers set phone = '13456789123' where id = '100001';

-- 这条语句能正确执行，因为phone列没有特殊的约束

-- 删除一个借阅者

delete from borrowers where id = '100003';

-- 这条语句能正确执行，borrow_info表中没有关联的记录

-- 删除所有借阅者

delete from borrowers;

-- 这条语句不能执行，如果borrow_info表中有关联的记录，因为会违反外键约束

-- 插入一些图书数据

insert into books values

('B001', '三国演义', '文学', false),

('B002', '高等数学', '理学', true),

('B003', '民法典', '法学', false),

('B004', '计算机网络', '工学', true),

('B005', '百科全书', '其他', false);

-- 这些数据能正确插入，因为它们满足了表的约束条件

-- 尝试插入一个不存在的类别

insert into books values

('B006', '人类简史：从动物到上帝', '历史', false);

-- 这条数据不能插入，因为category列只能是文学、理学、法学、工学或其他

-- 尝试插入一个重复的图书编号

insert into books values

('B003', '刑法典', '法学', true);

-- 这条数据不能插入，因为bookid列是主键，不能重复

-- 修改一个图书的名称

update books set bookname = '机器学习' where bookid = 'B004';

-- 这条语句能正确执行，因为bookname列没有特殊的约束

-- 修改一个图书的借出状态

update books set in_out = true where bookid = 'B001';

-- 插入一些借阅信息数据

insert into borrow_info (id, bookid, borrow_date, return_date) values

('100001', 'B001', '2023-05-01', null),

('100001', 'B002', '2023-05-10', null),

('100002', 'B004', '2023-05-15', null);

-- 这些数据能正确插入，因为它们满足了表的约束条件

-- 尝试插入一个不存在的借阅者编号

insert into borrow_info (id, bookid, borrow_date, return_date) values

('100004', 'B003', '2023-05-17',null);

-- 这条数据不能插入，因为id列是外键，要求在borrowers表中存在

-- 尝试插入一个不存在的图书编号

insert into borrow_info (id, bookid, borrow_date, return_date) values

('100001', 'B007', '2023-05-18', null);

-- 这条数据不能插入，因为bookid列是外键，要求在books表中存在

-- 尝试插入一个借阅日期晚于归还日期的数据

insert into borrow_info (id, bookid, borrow_date, return_date) values

('100003', 'B003', '2023-05-12', '2023-05-11');

-- 这条数据不能插入，因为borrow_info表有一个check约束，要求return_date >= borrow_date或return_date为空

-- 修改一个借阅信息的归还日期

update borrow_info set return_date = '2023-05-20' where id = '100001' and bookid = 'B001';

-- 这条语句能正确执行，因为return_date列没有特殊的约束

-- 修改一个借阅信息的到期日期

update borrow_info set due_date = '2023-06-01' where id = '100002' and bookid = 'B004';

-- 这条语句不能执行，因为due_date列是generated always as，不能修改

-- 删除一个借阅信息

delete from borrow_info where id = '100001' and bookid = 'B001';

-- 这条语句应该能正确执行，因为没有特殊的约束

-- 删除测试数据、插入完整数据

delete from borrow_info;

delete from books;

delete from borrowers;

insert into borrowers values

('100001', '张三', 'teacher', 2, '13812345678'),

('100002', '李四', 'student', 0, '13987654321'),

('100003', '王五', 'student', 1, '13765432198'),

('100004', '李老师', 'teacher', 1, '13987654321'),

('100005', '刘老师', 'teacher', 0, '13899998888'),

('100006', '陈老师', 'teacher', 1, '13966667777'),

('100007', '王同学', 'student', 3, '13611112222'),

('100008', '赵同学', 'student', 0, '13733334444'),

('100009', '孙同学', 'student', 2, '13655556666'),

('100010', '周同学', 'student', 1, '13744445555');

insert into books values

('B001', '三国演义', '文学', false),

('B002', '高等数学', '理学', true),

('B003', '民法典', '法学', false),

('B004', '计算机网络', '工学', true),

('B005', '百科全书', '其他', false),

('B006', '红楼梦', '文学', true),

('B007', '线性代数', '理学', false),

('B008', '刑法典', '法学', true),

('B009', '操作系统', '工学', false),

('B010', '世界地图集', '其他', true),

('B011', '水浒传', '文学', false),

('B012', '概率论与数理统计', '理学', true),

('B013', '国际法原理', '法学', false),

('B014', '数据结构与算法分析', '工学', true),

('B015', '世界历史大事记', '其他', false),

('B016', '西游记', '文学', false),

('B017', '微积分', '理学', true),

('B018', '宪法学', '法学', false),

('B019', '软件工程', '工学', true),

('B020', '世界名画欣赏', '其他', false),

('B021', '儒林外史', '文学', true),

('B022', '离散数学', '理学', false),

('B023', '民事诉讼法', '法学', true),

('B024', '编译原理', '工学', false),

('B025', '世界文化遗产导览', '其他', false);

insert into borrow_info (id,bookid,borrow_date,return_date) values

('100001', 'B001', '2023-04-01', '2023-04-15'),

('100001', 'B002', '2023-05-01', null),

('100001', 'B001', '2023-05-02', null),

('100003', 'B006', '2023-05-08', null),

('100004', 'B008', '2023-05-13', null),

('100006', 'B010', '2023-04-21', null),

('100007', 'B012', '2023-05-11', null),

('100007', 'B014', '2023-04-11', null),

('100007', 'B017', '2023-05-09', null),

('100009', 'B019', '2023-04-13', null),

('100009', 'B021', '2023-05-12', null),

('100010', 'B023', '2023-05-01', null);

2、存储过程测试

-- 假设张三想借《西游记》

call borrow_book('100001', 'B016');

-- 假设李四想借《高等数学》

call borrow_book('100002', 'B003');

-- 假设王五想借《高等数学》

call borrow_book('100003', 'B002');

-- 这里报错‘书籍已被借出’

-- 假设张三想还《三国演义》

call return_book('100001', 'B001');

-- 假设王五想还《红楼梦》

call return_book('100003', 'B006');

3、存储函数测试

#查询张三已借未还的图书情况

select bk.bookname,get_borrower_name(bk.bookid) as borrower_name from books bk join borrow_info bi on bk.bookid = bi.bookid

where bk.in_out = true and bi.return_date is null and get_borrower_name(bk.bookid) = '张三';



#显示所有逾期未归还图书的书名，借阅人和逾期时长并按逾期时长排序

select bk.bookname,b.name,get_overdue_days(bk.bookid) as 逾期时长

from borrow_info bi join books bk on bk.bookid = bi.bookid

join borrowers b on b.id = bi.id

where get_overdue_days(bk.bookid) > 0

order by 逾期时长 desc;



#调用函数显示有逾期未归还图书的借阅人和未归还图书数目

select name, get_overdue_books(id) as overdue_books

     from borrowers where get_overdue_books(id) > 0;



#调用该函数显示所有应缴纳罚款的借阅人的姓名，逾期罚款和电话

select name, get_fine(id) as fine, phone from borrowers

where get_fine(id) > 0;



4、触发器测试

-- 先查询王同学的借阅数量

select name,amount as 借阅数 from borrowers where id = '100007';



-- 再次查询王同学借阅数量

select name,amount as 借阅数 from borrowers where id = '100007';



-- 假设王同学还想借《编译原理》

call borrow_book('100007', 'B024');

-- 报错Error Code: 1644. Student cannot borrow more than 5 books

-- 假设王同学想还《数据结构与算法分析》

call return_book('100007', 'B014');

-- 再次查询王同学借阅数量

select name,amount as 借阅数 from borrowers where id = '100007';



--以上说明触发器均正常运行修改borrowers表中借阅人的借阅数量

​
