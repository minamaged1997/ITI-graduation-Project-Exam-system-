#the most Important stored Procedures used for the application.
create procedure [dbo].[GenerateExam] @Crs_ID int, @st_ID int
-- Stored procedure used in the login page in the desktop application
as
begin
	if @Crs_ID not in (select c.crs_id from Course c) 
  -- if the course ID entered is wrong*
		select 0
	else if @st_ID not in (select s.std_id from Student s )
	--if the student ID is wrong
		select 1
	else if @st_ID not in (select s.std_id from student_course s where s.crs_id=@Crs_ID)
	--if the student doesnt have this course in the curriculum
		select 2
	else if @Crs_ID in (select distinct q.crs_id from exam_Q_std e, Questions q where e.Q_ID=q.Q_id and e.std_id=@st_ID)
	-- if the student has already taken the exam on this course once
		select 3
	else 
		begin
		-- this piece code will generate 5 MCQ and 5 True or false questions randomly and insert them into the database then
		-- return them as a table to C# the application
			insert into exam_Q_std(Q_ID, std_id) select top (5) q.Q_id,@st_ID from Questions q where q.crs_id=@Crs_ID and q.Q_type='MCQ' order by NEWID()
			insert into exam_Q_std(Q_ID, std_id) select top (5) q.Q_id,@st_ID from Questions q where q.crs_id=@Crs_ID and q.Q_type='T/F' order by NEWID()

			select q.Q_id, q.Question, c.choice,q.Q_type, c.choice_id from exam_Q_std e, Questions q,Question_choices c where e.Q_ID=q.Q_id and q.Q_id=c.Q_id
			and e.std_id=@st_ID and q.crs_id=@Crs_ID order by q.Q_type
		end
end

--------------------------------------------------------------------------------------------------------------------------------------------------------

create procedure [dbo].[GetAnswers] 
@Stid int, @Answers as dbo.AnsType readonly
-- stored proc used to get the student answers from the desktop application and save it in the database 
-- we used cursor here to update the student answers in table (Exam_Q_std) which is saved in the database with
-- the new answers we get from the application.
as
begin
	declare t_cur cursor
		for select std_id, Q_ID from exam_Q_std
		for update

	declare @st_id int
	declare @Q_ID nvarchar(50)
	open t_cur 
	fetch t_cur into @st_id,@Q_ID
	begin
		While @@fetch_status=0
		begin
			if @st_id=@Stid and @Q_ID in (select a.Qid from @Answers a)
				update exam_Q_std	set std_answer= (select a.answer from @Answers a where a.Qid=@Q_ID)  where current of t_cur
			fetch t_cur into @st_id,@Q_ID
		end
	end
	close t_cur
	deallocate t_cur
end

-----------------------------------------------------------------------------------------------------------------------------------------------------------

create   procedure [dbo].[StudentResults](@st_Id int,@crs_id int, @grade int output)
as
-- stored procedure used to get student ID, Course ID and return the total grade after comparing student's answers with the Model answers. 
begin
	set nocount on 
	if not exists (select * from student where std_id = @st_Id)
		select 0
	else if not exists(select * from Course where crs_id = @crs_id)
		select 1
	else if not exists (select * from student_course where std_id = @st_Id and crs_id = @crs_id)
		select 2
	else if not exists (select * from exam_Q_std e , Questions q  where std_id = @st_Id and crs_id = @crs_id)
		select 3

	select e.Q_ID,std_id,crs_id,
	case 
		when e.std_answer = q.model_answer 
			then  Q_grade
		else
			0
		end as last_grade into #newtable
	from exam_Q_std e , Questions q
	where e.Q_ID = q.Q_id and std_id = @st_Id and crs_id = @crs_id

	update exam_Q_std
	set std_Q_grade = last_grade , exam_corrected = 1
	from exam_Q_std e , Questions q , #newtable n 
	where e.Q_ID = n.Q_ID and e.std_id = @st_Id and q.crs_id = @crs_id

	update student_course set grade_overall=
	( select SUM(last_grade) as Total_degrees from #newtable
	where crs_id=@crs_id and std_id=@st_Id)
	where std_id=@st_Id and crs_id=@crs_id

	set @grade= (select grade_overall from student_course where std_id=@st_Id and crs_id=@crs_id)

end

