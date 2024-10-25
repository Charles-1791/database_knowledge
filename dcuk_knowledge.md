# Optimizer Entrance
in client_context.cpp: ClientContext::CreatePreparedStatmentmentInternal
![image](https://github.com/user-attachments/assets/5cb954f5-4217-43e3-b828-21891a29160f)

optimizer.Optimize is called on the input plan.
Then RunBuiltInOptimizers() is then called, in which a series of optimizers are called consecutively
![image](https://github.com/user-attachments/assets/5188aad0-428e-4d46-8e47-eb753f16679b)

most of these optimizers have a function redirecting the current plan node to different handler based on the node type.
![image](https://github.com/user-attachments/assets/56e9f320-2453-430a-8f1c-96773927d192)

# Logical Operator
LogicalOperator is the base class for plan nodes, such as LogicalFilter and LigicalProjection. It is similar to the logicalPlanBase under our framework.

## ColumnBinding
```
struct ColumnBinding {
	idx_t table_index;
	// This index is local to a Binding, and has no meaning outside of the context of the Binding that created it
	idx_t column_index;
  ...
}
```
Looks like a pair of index like <table_index, column_index>, is used to specify a column in a plan, similar to our unique id.
```
virtual vector<ColumnBinding> GetColumnBindings();
```
This virtual function is override by any concrete logical nodes inheriting from it, and it returns the output columns offered by current node. (Similar to our OutputCol)


## Cast()
A commonly used function template is :
```
template <class TARGET>
TARGET &Cast() {
  if (TARGET::TYPE != LogicalOperatorType::LOGICAL_INVALID && type != TARGET::TYPE) {
    throw InternalException("Failed to cast logical operator to type - logical operator type mismatch");
  }
  return reinterpret_cast<TARGET &>(*this);
}
```
And it is just a downcasting of the current operator to a concrete plan type.

# Plan BUilder 
The following function takes in a statment and returns the raw plan.
```
void Planner::CreatePlan(unique_ptr<SQLStatement> statement) {
	D_ASSERT(statement);
	switch (statement->type) {
	case StatementType::SELECT_STATEMENT:
	case StatementType::INSERT_STATEMENT:
	case StatementType::COPY_STATEMENT:
	case StatementType::DELETE_STATEMENT:
	case StatementType::UPDATE_STATEMENT:
	case StatementType::CREATE_STATEMENT:
	case StatementType::DROP_STATEMENT:
	case StatementType::ALTER_STATEMENT:
	case StatementType::TRANSACTION_STATEMENT:
	case StatementType::EXPLAIN_STATEMENT:
	case StatementType::VACUUM_STATEMENT:
	case StatementType::RELATION_STATEMENT:
	case StatementType::CALL_STATEMENT:
	case StatementType::EXPORT_STATEMENT:
	case StatementType::PRAGMA_STATEMENT:
	case StatementType::SET_STATEMENT:
	case StatementType::LOAD_STATEMENT:
	case StatementType::EXTENSION_STATEMENT:
	case StatementType::PREPARE_STATEMENT:
	case StatementType::EXECUTE_STATEMENT:
	case StatementType::LOGICAL_PLAN_STATEMENT:
	case StatementType::ATTACH_STATEMENT:
	case StatementType::DETACH_STATEMENT:
	case StatementType::COPY_DATABASE_STATEMENT:
	case StatementType::UPDATE_EXTENSIONS_STATEMENT:
		CreatePlan(*statement);
		break;
	default:
		throw NotImplementedException("Cannot plan statement of type %s!", StatementTypeToString(statement->type));
	}
}
```

